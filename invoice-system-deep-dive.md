# Invoice System — End-to-End Deep Dive

This document traces every request, service call, database write, and event from the moment a vendor or enterprise admin touches "Add invoice" through to payment execution. All explanations are anchored to the exact flow diagram (`invoice flow.png`) in this directory.

---

## The Flow Diagram (summary)

```
Invoice submitted (vendor or customer)
  ↓
Finance Executive → "READY FOR PAYMENT"
  ↓
Finance Controller → "APPROVED FOR PAYMENT on payment_date"
  ↓
Spend Service receives event: { invoice_amt, payment_date, payment_amt, note }
  note = set when payment_amt < invoice_amt (partial payment)
  ↓
  ├─ If payment_date = today  → process immediately
  └─ If payment_date = future → BullMQ delayed job
                                + pendingCreditPaise += payment_amt (node + all parents)
                                + CategoryLimit.pendingPaise += payment_amt

(on payment_date)
Check BOTH limits across node tree:
  - CategoryLimit[category]: limitPaise - usedPaise - (pendingPaise - payment_amt) >= payment_amt
  - OrgNode.creditLimit: same pattern up to root/enterprise
  ↓
  ├─ REJECTED → Invoice status = REJECTED
  └─ Create LMS drawdown at payment_amt
       usedCreditPaise   += payment_amt (node + all parents)
       pendingCreditPaise -= payment_amt (node + all parents)
       CategoryLimit.usedPaise    += payment_amt
       CategoryLimit.pendingPaise -= payment_amt
       Invoice status = APPROVED (or PARTIALLY_PAID if payment_amt < invoice_amt)
```

---

## File Map

| File | Role |
|---|---|
| `bsms.controller.ts` | HTTP entry point — all external invoice endpoints |
| `invoice-analysis-callback.controller.ts` | Internal HTTP entry point — AI agent + worker callbacks |
| `invoice.service.ts` | Orchestrates FM/CFO/EA/FC invoice actions, drawdown, scheduled payments |
| `vendor-invoice.service.ts` | Vendor-scoped operations — submit invoice, upload PDF |
| `invoice.repository.ts` | All DB reads/writes for the Invoice table |
| `invoice-audit.repository.ts` | Append-only audit trail writes |
| `vendor-registry.repository.ts` | Reads VendorRegistry — validates vendor membership |
| `invoice-limit-engine.ts` | Interface + stub for OrgNode limit checks (spend team owns real impl) |
| `invoice-storage.service.ts` | GCS signed URL generation for PDF upload/download |
| `invoice-sse.service.ts` | Real-time push — routes domain events to SSE connections |
| `invoice-dispute.service.ts` | Dispute raise/resolve lifecycle |
| `invoice-dispute.repository.ts` | DB reads/writes for InvoiceDispute table |
| `bull-mq-shutdown.service.ts` | Graceful SIGTERM — drains BullMQ queues before exit |
| `bsms.events.ts` | All domain event classes for the invoice flow |
| `bsms.module.ts` | NestJS wiring — providers, queues, injection tokens |

---

## Journey 1 — Vendor submits an invoice

### Step 1 — Frontend

Vendor opens their portal and clicks **Add invoice**. Frontend calls:

```
POST /bsms/invoices/vendor-upload-url
```
to get a signed GCS upload URL, uploads the PDF directly to GCS, then calls:

```
POST /bsms/invoices/vendor
Body: { vendorRegistryId, spendCategory, currency, s3Key }
```

### Step 2 — Controller (`bsms.controller.ts`)

```typescript
@Post('invoices/vendor')
@Roles(Role.VENDOR)
async vendorSubmitInvoiceAuthenticated(@Body() dto, @CurrentUser() user)
```

Validates with an inline `z.object({...})` Zod schema (anonymous — not a named const). Calls:

```typescript
this.vendorInvoiceSvc.submitInvoice(user.userId, dto)
```

### Step 3 — `VendorInvoiceService.submitInvoice`

**File:** `vendor-invoice.service.ts`

1. `vendorRegistryRepo.findVerifiedByUserId(userId)` — loads all VendorRegistry rows the vendor owns. Confirms `dto.vendorRegistryId` is in that list. If not → throws `VendorRegistryNotOwnedError`.

2. Creates invoice row via `invoiceRepo.createInvoice({...})` with:
   - `invoiceRef = "DRAFT-<uuid>"` — placeholder, AI will overwrite this
   - `amountMinorUnit = 0` — placeholder, AI will overwrite this
   - `status = SUBMITTED`
   - `source = VENDOR_SUBMITTED`

3. Writes `SUBMITTED` audit event via `auditRepo.create`.

4. Publishes `InvoiceSubmittedEvent` to the event bus.

5. Gets a 1-hour signed GCS read URL for the PDF (so the Python AI agent can fetch it via HTTPS).

6. Calls `invoiceRepo.atomicSetAiProcessing(invoice.id, enterpriseId)` — this is a **single Prisma transaction** that:
   - Updates `invoice.status = AI_PROCESSING`
   - Writes `AI_PROCESSING` audit event in the same transaction
   
   The DB-update-before-enqueue order is intentional: if BullMQ enqueue fails after this, the invoice is visibly stuck in `AI_PROCESSING` and can be re-queued. The reverse order would leave a live job pointing at a `SUBMITTED` invoice with no recovery path.

7. Enqueues the `analyse` job on `INVOICE_ANALYSIS_QUEUE`:
   ```json
   {
     "invoiceId":       "...",
     "enterpriseId":    "...",
     "vendorRegistryId":"...",
     "invoiceRef":      "DRAFT-XXXXXXXX",
     "invoiceDate":     "2026-06-06T00:00:00.000Z",
     "amountMinorUnit": "0",
     "currency":        "INR",
     "pdfUrl":          "https://signed-gcs-url-or-null"
   }
   ```
   BullMQ retries 3× with exponential backoff (5s base).

8. Returns `{ ...invoice, status: AI_PROCESSING }`.

### Step 4 — AI Agent (Python worker, separate process)

The Python invoice-agent picks up the job from `INVOICE_ANALYSIS_QUEUE`, fetches the PDF, runs OCR + GSP (GST Portal) match, and POSTs the result back to:

```
POST /internal/invoices/:id/analysis-complete
```

This is protected by `InternalGuard` — only reachable from within the same VPC.

### Step 5 — `InvoiceAnalysisCallbackController.analysisComplete`

**File:** `invoice-analysis-callback.controller.ts`

Validates the body with `AnalysisCompleteSchema`:
```typescript
{
  status:          'AI_VALIDATED' | 'AI_FLAGGED' | 'AI_FAILED'
  extractedFields: Record<string, unknown> | null
  gspMatchStatus:  'MATCHED' | 'FLAGGED' | 'SKIPPED' | null
  aiFlags:         string[]
}
```

Calls `invoiceService.handleAnalysisComplete(id, dto)`.

### Step 6 — `InvoiceService.handleAnalysisComplete`

**File:** `invoice.service.ts`

1. Loads invoice. Throws if not found or not in `AI_PROCESSING` — guards against duplicate callbacks.

2. Calls `invoiceRepo.updateAiAnalysis(id, { status, extractedFields, gspMatchStatus, aiFlags })` — sets the AI result columns.

3. If `extractedFields` has values, patches the main invoice columns:
   - `invoiceRef` ← `ef.invoice_ref` (replaces the `DRAFT-xxx` placeholder)
   - `invoiceDate` ← `ef.invoice_date`
   - `amountMinorUnit / totalAmountPaise / amountInr` ← `ef.total_paise`
   - Cap: rejects amounts > ₹50 lakh (500,000,000 paise) to guard against hallucinated OCR
   - On duplicate `invoiceRef` (P2002): retries patch without the ref so amount/date still update

4. Writes audit event: `AI_VALIDATED`, `AI_FLAGGED`, or `AI_FAILED`.

5. Publishes `InvoiceAiAnalysisCompletedEvent` → SSE push to all connected clients for this enterprise.

**Result after Step 6:** Invoice is now `AI_VALIDATED` or `AI_FLAGGED`. It appears in the FE/EA verify queue.

---

## Journey 2 — Enterprise Admin manually enters an invoice

EA opens **Add invoice for vendor**. Frontend calls `POST /bsms/invoices/enterprise-for-vendor`.

### Controller → `InvoiceService.submitForVendor`

1. `vendorRegistryRepo.findVerifiedByIdAndEnterprise(vendorRegistryId, enterpriseId)` — validates vendor belongs to this enterprise.

2. Creates invoice via `invoiceRepo.createInvoice` with:
   - `status = AI_VALIDATED` — skips AI entirely, EA provided all fields
   - `source = ENTERPRISE_ENTERED`
   - Real `invoiceRef`, `invoiceDate`, `amountMinorUnit` from the form

3. Writes `SUBMITTED` audit event.

4. Publishes `InvoiceSubmittedEvent`.

**Result:** Invoice lands directly at `AI_VALIDATED` — straight into the FE/EA verify queue. No AI processing step.

---

## Journey 3 — Finance Executive verifies

FE sees the invoice in their pending queue (status `AI_VALIDATED` or `AI_FLAGGED`). They select a budget node and click **Verify invoice**.

```
POST /bsms/invoices/:id/verify
Body: { orgNodeId }
```

### Controller → `InvoiceService.verifyInvoice`

**File:** `invoice.service.ts`

1. Loads invoice. Guards: must exist, must belong to this enterprise, must be `AI_VALIDATED` or `AI_FLAGGED`.

2. Extracts `invoice.amountMinorUnit` as `effectiveAmount` and `invoice.spendCategory` as `category`.

3. Calls `runVerifyLimitCheck(this.limitEngine, orgNodeId, category, effectiveAmount)`.

### `runVerifyLimitCheck` (in `invoice-limit-engine.ts`)

This function runs two checks and returns the resulting status:

```typescript
// Check 1: credit limit (node-level, category-agnostic)
const credit = await engine.checkCreditLimit(orgNodeId, amountPaise);
if (!credit.passed) return InvoiceStatus.REJECTED;

// Check 2: spend limit (category-specific)
const spend = await engine.checkSpendLimit(orgNodeId, category, amountPaise);
if (!spend.passed) return InvoiceStatus.SPEND_LIMIT_FLAGGED;

return InvoiceStatus.READY_FOR_PAYMENT;
```

**What each check means:**

- `checkCreditLimit`: Walks the OrgNode tree from `orgNodeId` to root. At each node checks `allocatedAmount - usedAmount - pendingAmount >= amountPaise`. This is the hard credit wall — fail means the enterprise's LOC cannot cover this at all → REJECTED.

- `checkSpendLimit`: Checks `CategoryLimit` for this category at each node. Formula: `limitPaise - usedPaise - (pendingPaise - amountPaise) >= amountPaise`. The `- amountPaise` subtraction removes this invoice's own pending reservation from the formula (avoids double-counting when re-checking on payment_date).

**Results from `runVerifyLimitCheck`:**

| Result | Meaning | Next status |
|---|---|---|
| `REJECTED` | Credit limit exceeded | `REJECTED` |
| `SPEND_LIMIT_FLAGGED` | Credit OK, but category budget insufficient for full amount | `SPEND_LIMIT_FLAGGED` |
| `READY_FOR_PAYMENT` | Both limits clear | `READY_FOR_PAYMENT` |

4. Back in `verifyInvoice`:
   - If `REJECTED`: updates status, writes rejection audit, publishes `InvoiceRejectedEvent`.
   - Otherwise: updates status to `READY_FOR_PAYMENT` or `SPEND_LIMIT_FLAGGED`, writes `VERIFIED` audit with note `"Spend limit breached — forwarded to FC"` if flagged, publishes `InvoiceVerifiedEvent`.

5. `orgNodeId` is stored on the invoice at this point — every subsequent step uses it.

---

## Journey 4 — Finance Controller schedules payment

FC sees the invoice in their queue (status `READY_FOR_PAYMENT` or `SPEND_LIMIT_FLAGGED`). For `SPEND_LIMIT_FLAGGED`, the UI shows a callout explaining the node's category budget is insufficient and a partial payment will be auto-computed. FC sets only the **payment date** and clicks **Approve for payment**.

```
POST /bsms/invoices/:id/approve-for-payment
Body: { paymentDate: "2026-06-10" }
```

### Controller → `InvoiceService.approveForPayment`

**File:** `invoice.service.ts`

1. Loads invoice. Guards: must be `READY_FOR_PAYMENT` or `SPEND_LIMIT_FLAGGED`.

2. **Auto-compute `payment_amt`** (this is the server-side auto-computation — FC never types an amount):

   ```
   READY_FOR_PAYMENT:   payment_amt = invoice_amt  (full payment, limits were clear at FE step)
   SPEND_LIMIT_FLAGGED: payment_amt = min(getAvailableSpend(), invoice_amt)  (partial)
   ```

   For `SPEND_LIMIT_FLAGGED`, calls `limitEngine.getAvailableSpend(orgNodeId, category)` which returns `CategoryLimit.limitPaise - usedPaise - pendingPaise` for this node.

3. **Compute `note`**: If `payment_amt < invoice_amt` (partial), sets `note = "Partial payment: X of Y paise"`. Otherwise `null`. This matches the flow diagram: `note = used when payment_amt < invoice_amt`.

4. **Routing by date:**

   **Today (payment_date ≤ today):**
   - Calls `runPaymentDrawdown(invoice, payment_amt, userId, enterpriseId, note)` immediately.

   **Future date:**
   - Calls `limitEngine.reserveNodeCredit(orgNodeId, category, payment_amt)`:
     ```
     OrgNode.pendingAmount += payment_amt  (orgNodeId + all parents up to root)
     CategoryLimit.pendingPaise += payment_amt
     ```
     This is the "reservation" the flow diagram calls out: `pendingCreditPaise += payment_amt`.
   - Updates invoice status to `PAYMENT_SCHEDULED`, stores `scheduledPaymentDate` and `scheduledPaymentAmtPaise` on the row.
   - Enqueues a delayed BullMQ job on `INVOICE_PAYMENT_QUEUE`:
     ```json
     { "invoiceId": "...", "enterpriseId": "..." }
     ```
     `delay = payment_date.getTime() - Date.now()` — BullMQ fires the job at exactly that millisecond.
   - Writes `PAYMENT_SCHEDULED` audit event with `note` and `amountPaise`.
   - Publishes `InvoicePaymentScheduledEvent` (carries `payment_amt` and `note`).

---

## Journey 5 — BullMQ fires on payment_date

When the scheduled job fires, the BullMQ worker (in `apps/worker`) makes an HTTP call to:

```
POST /internal/invoices/:id/process-payment
Body: { enterpriseId }
```

### `InvoiceAnalysisCallbackController.processPayment`

Calls `invoiceService.processScheduledPayment(id, enterpriseId)`.

### `InvoiceService.processScheduledPayment`

**File:** `invoice.service.ts`

This is **idempotent** — if the invoice is no longer `PAYMENT_SCHEDULED` (already processed or manually rejected), it writes a `PAYMENT_SKIPPED` audit event and returns.

1. Loads `payment_amt` from `invoice.scheduledPaymentAmtPaise` (stored at FC approval time). Falls back to `invoice.amountMinorUnit` if null — this is a data-integrity guard only.

2. **Re-checks BOTH limits** (flow diagram: "Check BOTH limits across node tree on payment_date"):

   **Check A — credit limit:**
   ```typescript
   const credit = await this.limitEngine.checkCreditLimit(orgNodeId, payment_amt);
   ```
   Formula: `OrgNode.allocatedAmount - usedAmount - (pendingAmount - payment_amt) >= payment_amt`
   
   The `- payment_amt` inside pendingAmount subtracts this invoice's own reservation (placed at FC approval) so the check doesn't count it as competing demand. This is the exact formula from the flow diagram.

   If fails → `limitEngine.releaseNodeCredit(orgNodeId, category, payment_amt)`:
   ```
   OrgNode.pendingAmount -= payment_amt  (orgNodeId + all parents)
   CategoryLimit.pendingPaise -= payment_amt
   ```
   Then calls `_rejectScheduledPayment` → status `REJECTED`, audit event, `InvoiceRejectedEvent`.

   **Check B — spend limit:**
   ```typescript
   const spend = await this.limitEngine.checkSpendLimit(orgNodeId, category, payment_amt);
   ```
   Same formula but on `CategoryLimit`. Handles the case where the category budget was tightened between FE verify and payment_date (another invoice consumed the budget in the interim).

   If fails → same `releaseNodeCredit` + `_rejectScheduledPayment` path.

3. If both pass → calls `runPaymentDrawdown(invoice, payment_amt, null, enterpriseId, null)`.

4. Writes `PAYMENT_PROCESSED` audit event.

---

## Journey 6 — `runPaymentDrawdown` (the actual payment)

**File:** `invoice.service.ts` — private method, called from three places:
- `approveInvoice` (FM/CFO direct approve, today)
- `approveForPayment` when payment_date = today
- `processScheduledPayment` when BullMQ fires

### What it does

1. **LMS drawdown** — calls `lmsGateway.createDrawdown(drawdownInput)`:
   ```typescript
   {
     enterpriseId,
     invoiceId:   invoice.id,
     invoiceRef:  invoice.invoiceRef,
     amountPaise: payment_amt,  // = payment_amt from flow diagram
     currency:    invoice.currency,
   }
   ```
   LMS = Frappe Lending (Hyperface in the diagram). This creates the actual drawdown against the enterprise's LOC. Returns `{ drawdownRef, status }`.

2. **Settle OrgNode limits** (flow diagram: "Recalculate limits across node tree"):
   ```typescript
   await this.limitEngine.settleNodeSpend(orgNodeId, spendCategory, payment_amt);
   ```
   What this does:
   ```
   OrgNode.usedAmount    += payment_amt  (orgNodeId + all parents up to root)
   OrgNode.pendingAmount -= payment_amt  (orgNodeId + all parents up to root)
   CategoryLimit.usedPaise    += payment_amt
   CategoryLimit.pendingPaise -= payment_amt
   ```
   Converts the pending reservation into settled spend.

3. **Determine final status**:
   ```typescript
   const isPartial   = payment_amt < invoice.amountInr;
   const finalStatus = isPartial ? InvoiceStatus.PARTIALLY_PAID : InvoiceStatus.APPROVED;
   ```

4. **Updates invoice** to `APPROVED` or `PARTIALLY_PAID`, stores:
   - `lmsDrawdownRef` — reference from LMS for reconciliation
   - `approvedAmountPaise` — what was actually paid (= `payment_amt`, may be less than invoice total)
   - `reviewedAt`, `reviewedByUserId` (null if called from worker)

5. **Writes audit event** `APPROVED` with `note` (non-null when partial) and `amountPaise = payment_amt`.

6. **Publishes `InvoiceApprovedEvent`** — carries `drawdownRef` and `payment_amt`.

---

## Reject paths

### Manual reject (FM/CFO/EA/FE)

```
POST /bsms/invoices/:id/reject
Body: { rejectionReason }
```

→ `InvoiceService.rejectInvoice`

Guard: cannot reject if status is in `TERMINAL_STATUSES`. This includes:
- `READY_FOR_PAYMENT` / `SPEND_LIMIT_FLAGGED` — FE has already verified; these must proceed to FC, not be rejected via this endpoint.
- `PAYMENT_SCHEDULED` — BullMQ job exists; rejecting would orphan it.
- `APPROVED`, `PARTIALLY_PAID`, `REJECTED`, `CANCELLED`, `CONVERTED`, `PAID` — already terminal.

Writes `REJECTED` status, audit event, publishes `InvoiceRejectedEvent`.

### Scheduled payment rejected at execution

Handled inside `processScheduledPayment` → `_rejectScheduledPayment`. Always calls `releaseNodeCredit` first to undo the pending reservation before rejecting.

---

## Supporting Services

### `InvoiceStorageService` (`invoice-storage.service.ts`)

Wraps Google Cloud Storage. Two operations:

- `getSignedUploadUrl(enterpriseId, filename, contentType)` — returns a write-signed URL (15 min) + GCS path `invoices/{enterpriseId}/{uuid}/{filename}`. Vendor browser uploads directly to GCS — API never receives the file bytes.
- `getSignedReadUrl(gcsPath, expiresIn)` — returns a read-signed URL. Used when FM/CFO/FE clicks "View PDF" in the invoice sheet. Returns `null` in stub mode (no GCP credentials configured).

### `InvoiceSseService` (`invoice-sse.service.ts`)

Subscribes to `eventBus` with a wildcard `*` listener on boot. Filters for `eventType.startsWith('invoice.')`. For each matching event, emits to an `EventEmitter` keyed by `enterpriseId`.

Frontend keeps a persistent `GET /bsms/invoices/events` connection. When any invoice event fires for that enterprise (submit, AI complete, verify, approve, reject, schedule), the SSE stream delivers `{ type, invoiceId }` and the frontend re-fetches the invoice list — no polling needed.

Heartbeat: sends an empty `data: ''` frame every 25s to prevent proxy/load-balancer idle timeout (Nginx default is 60s).

### `InvoiceAuditRepository` (`invoice-audit.repository.ts`)

Append-only. Every state transition writes a row to `InvoiceAuditEvent`:
- `action` — enum value from `InvoiceAuditAction`
- `actorName` / `actorRole` — snapshot at time of action (never a join — person's name/role could change)
- `note` — free text, used for rejection reasons, partial payment amounts, skip reasons
- `amountPaise` — the amount involved in the action (null for verify/reject without amount)

`resolveUser(userId)` — best-effort lookup of `User.name` + `User.role` for the audit snapshot. Never throws — returns `{ name: null, role: null }` on any error.

### `IInvoiceLimitEngine` (`invoice-limit-engine.ts`)

Interface with injection token `INVOICE_LIMIT_ENGINE`. Currently wired to `InvoiceLimitEngineStub` in `bsms.module.ts`. The stub:
- All checks return `{ passed: true }` — nothing is blocked in dev/staging
- All mutations are no-ops
- `getAvailableSpend` returns `BigInt(Number.MAX_SAFE_INTEGER)` — so `min(available, invoice_amt) = invoice_amt` always = full payment

To switch to the real implementation:
```typescript
// bsms.module.ts providers:
{ provide: INVOICE_LIMIT_ENGINE, useClass: RealInvoiceLimitEngine }
```

The spend team owns `RealInvoiceLimitEngine`. The interface contract is fixed.

### `BullMqShutdownService` (`bull-mq-shutdown.service.ts`)

Listens for `SIGTERM`. On shutdown, drains `INVOICE_ANALYSIS_QUEUE` and `INVOICE_PAYMENT_QUEUE` and closes the shared Redis connection. Prevents jobs from being mid-flight when the process exits.

---

## Domain Events (bsms.events.ts)

| Event class | eventType | Fired when |
|---|---|---|
| `InvoiceSubmittedEvent` | `invoice.submitted` | Vendor or EA creates invoice |
| `InvoiceAiAnalysisCompletedEvent` | `invoice.ai_analysis.completed` | AI agent posts result |
| `InvoiceVerifiedEvent` | `invoice.verified` | FE/EA verifies invoice |
| `InvoiceRejectedEvent` | `invoice.rejected` | Any rejection path |
| `InvoicePaymentScheduledEvent` | `invoice.payment_scheduled` | FC schedules future payment |
| `InvoiceApprovedEvent` | `invoice.approved` | Payment drawdown succeeds |
| `InvoiceDisputeRaisedEvent` | `invoice.dispute.raised` | Vendor raises dispute |
| `InvoiceDisputeResolvedEvent` | `invoice.dispute.resolved` | FM/CFO resolves dispute |

All events carry `enterpriseId` and `invoiceId` so downstream consumers can identify what changed.

---

## Invoice Status State Machine

```
SUBMITTED
  └─ AI_PROCESSING  (atomicSetAiProcessing)
       ├─ AI_VALIDATED   (handleAnalysisComplete)
       ├─ AI_FLAGGED     (handleAnalysisComplete)
       └─ AI_FAILED      (handleAnalysisComplete)

AI_VALIDATED / AI_FLAGGED
  ├─ REJECTED            (rejectInvoice OR verifyInvoice → credit limit fail)
  ├─ SPEND_LIMIT_FLAGGED (verifyInvoice → spend limit fail, credit OK)
  └─ READY_FOR_PAYMENT   (verifyInvoice → both limits clear)

READY_FOR_PAYMENT / SPEND_LIMIT_FLAGGED
  ├─ APPROVED            (approveForPayment → today → drawdown → full)
  ├─ PARTIALLY_PAID      (approveForPayment → today → drawdown → partial)
  └─ PAYMENT_SCHEDULED   (approveForPayment → future date)
  (rejectInvoice BLOCKED — both statuses are in TERMINAL_STATUSES)

PAYMENT_SCHEDULED
  ├─ REJECTED            (processScheduledPayment → limit check fail)
  ├─ APPROVED            (processScheduledPayment → drawdown → full)
  └─ PARTIALLY_PAID      (processScheduledPayment → drawdown → partial)

APPROVED / PARTIALLY_PAID / REJECTED → terminal (no further transitions)
```

`TERMINAL_STATUSES` in `invoice.service.ts` blocks any action on these statuses. `PAYMENT_SCHEDULED` is also in the set — once a BullMQ job exists, manual rejection is blocked to prevent orphaned jobs.

---

## All amounts are paise as BigInt

Every amount in the system (DB columns, service variables, event fields) is stored and computed as `BigInt` paise. `invoice_amt` and `payment_amt` in the flow diagram map directly to `invoice.amountInr` and `payment_amt` BigInt variables in the service. Never float, never Decimal, never rupees inside the service layer.
