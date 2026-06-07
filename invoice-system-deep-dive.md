# Invoice System — End-to-End Deep Dive

This document traces every request, service call, database write, and event from the moment a vendor or enterprise admin touches "Add invoice" through to payment execution. All explanations are anchored to the exact flow diagram (`invoice flow.png`) in this directory.

Last updated: 2026-06-07 (NEXO-156)

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
                                + VelocityCounter.pendingAmount += payment_amt (all nodes)
                                + OrgNode.pendingAmount += payment_amt ← ARCHITECT GAP (not yet written)

(on payment_date)
Check BOTH limits across node tree:
  - CategoryLimit[category]: usedAmount + pendingAmount <= limitAmount
  - OrgNode credit limit: usedAmount + pendingAmount <= allocatedAmount
  ↓
  ├─ REJECTED → releaseNodeCredit → Invoice status = REJECTED
  └─ Create LMS drawdown at payment_amt
       VelocityCounter.usedAmount    += payment_amt (all nodes)
       VelocityCounter.pendingAmount -= payment_amt (all nodes)
       OrgNode.usedAmount    += payment_amt ← ARCHITECT GAP (not yet written)
       OrgNode.pendingAmount -= payment_amt ← ARCHITECT GAP (not yet written)
       Invoice status = APPROVED (or PARTIALLY_PAID if payment_amt < invoice_amt)
```

---

## File Map

| File | Role |
|---|---|
| `bsms.controller.ts` | HTTP entry point — all external invoice endpoints |
| `invoice-analysis-callback.controller.ts` | Internal HTTP entry point — AI agent + worker callbacks |
| `invoice.service.ts` | Orchestrates FM/CFO/EA/FC invoice actions, drawdown, scheduled payments |
| `vendor-invoice.service.ts` | Vendor-scoped operations — submit invoice, upload/validate PDF |
| `invoice.repository.ts` | All DB reads/writes for the Invoice table |
| `invoice-audit.repository.ts` | Append-only audit trail writes |
| `vendor-registry.repository.ts` | Reads VendorRegistry — validates vendor membership |
| `invoice-limit-engine.ts` | Interface + real impl + stub for OrgNode limit checks |
| `invoice-storage.service.ts` | GCS signed URL generation for PDF upload/download |
| `invoice-sse.service.ts` | Real-time push — routes domain events to SSE connections |
| `invoice-dispute.service.ts` | Dispute raise/resolve lifecycle |
| `invoice-dispute.repository.ts` | DB reads/writes for InvoiceDispute table |
| `bull-mq-shutdown.service.ts` | Graceful SIGTERM — drains BullMQ queues before exit |
| `bsms.events.ts` | All domain event classes for the invoice flow |
| `bsms.module.ts` | NestJS wiring — providers, queues, injection tokens |

---

## Journey 1 — Vendor submits an invoice

### Step 1a — PDF upload (required)

Vendor clicks **Upload PDF**. Frontend calls:

```
POST /bsms/invoices/vendor-upload
Content-Type: multipart/form-data
Body: file (PDF), vendorRegistryId
```

→ `VendorInvoiceService.uploadInvoicePdf`

API receives file bytes server-side. **No signed URL path for vendors** — server-side upload only.

**Validation before storing:**
- MIME type must be `application/pdf` → `InvoiceInvalidTypeError`
- File size ≤ 10 MB (enforced by multer limit before handler runs) → `InvoiceFileTooLargeError`
- Password-protected PDF → `InvoicePasswordProtectedError` (detected via `pdf-parse` — tries to parse page 1; catches `PasswordException` from pdfjs before any content extraction)

Saves to GCS via `invoiceStorage.save()`. Returns `{ s3Key }` — GCS path `invoices/{enterpriseId}/{uuid}/{filename}`.

### Step 1b — Submit invoice

`s3Key` from Step 1a is **required** in the submit body (controller Zod schema: `z.string().min(1)`, not optional).

Frontend calls:

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

Validates with inline Zod schema. Calls:

```typescript
this.vendorInvoiceSvc.submitInvoice(user.userId, dto)
```

### Step 3 — `VendorInvoiceService.submitInvoice`

**File:** `vendor-invoice.service.ts`

1. `vendorRegistryRepo.findVerifiedByUserId(userId)` — loads all VendorRegistry rows the vendor owns. Confirms `dto.vendorRegistryId` is in that list. If not → throws `VendorRegistryNotOwnedError`.

2. Creates invoice row via `invoiceRepo.createInvoice({...})` with:
   - `invoiceRef = "DRAFT-<8-char-uuid>"` — placeholder, AI overwrites this
   - `amountMinorUnit = 0` — placeholder, AI overwrites this
   - `status = SUBMITTED`
   - `source = VENDOR_SUBMITTED`

3. Writes `SUBMITTED` audit event via `auditRepo.create`.

4. Publishes `InvoiceSubmittedEvent` to the event bus.

5. Gets a 1-hour signed GCS read URL for the PDF (so the Python AI agent can fetch it via HTTPS). If no `s3Key`, `pdfUrl = null`.

6. Calls `invoiceRepo.atomicSetAiProcessing(invoice.id, enterpriseId)` — **single Prisma transaction** that:
   - Updates `invoice.status = AI_PROCESSING`
   - Writes `AI_PROCESSING` audit event in the same transaction

   DB-update-before-enqueue order is intentional: if BullMQ enqueue fails after this, invoice is visibly stuck in `AI_PROCESSING` and can be re-queued. Reverse order would leave a live job pointing at a `SUBMITTED` invoice with no recovery path.

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

Python invoice-agent picks up the job from `INVOICE_ANALYSIS_QUEUE`, fetches the PDF, runs OCR + GSP (GST Portal) match, POSTs result back to:

```
POST /internal/invoices/:id/analysis-complete
```

Protected by `InternalGuard` — only reachable from within the same VPC.

### Step 5 — `InvoiceAnalysisCallbackController.analysisComplete`

**File:** `invoice-analysis-callback.controller.ts`

Validates body with `AnalysisCompleteSchema`:
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

2. Calls `invoiceRepo.updateAiAnalysis(id, { status, extractedFields, gspMatchStatus, aiFlags })`.

3. If `extractedFields` has values, patches main invoice columns:
   - `invoiceRef` ← `ef.invoice_ref` (replaces `DRAFT-xxx`)
   - `invoiceDate` ← `ef.invoice_date`
   - `amountMinorUnit / totalAmountPaise / amountInr` ← `ef.total_paise`
   - Cap: rejects amounts > ₹50 lakh (500,000,000 paise) against hallucinated OCR
   - On duplicate `invoiceRef` (P2002): retries patch without the ref so amount/date still update

4. Writes audit event: `AI_VALIDATED`, `AI_FLAGGED`, or `AI_FAILED`.

5. Publishes `InvoiceAiAnalysisCompletedEvent` → SSE push to all connected clients.

**Result:** Invoice is now `AI_VALIDATED` or `AI_FLAGGED`. Appears in FE/EA verify queue.

---

## Journey 2 — Enterprise Admin manually enters an invoice

EA can optionally upload a PDF first using the signed URL path (enterprise-only):

```
POST /bsms/invoices/enterprise-upload-url
Body: { vendorRegistryId, filename, contentType }
→ returns { uploadUrl, s3Key, expiresIn: 900 }
```

EA browser uploads PDF directly to GCS using `uploadUrl`. Keeps `s3Key`.

**Known gap:** `POST /bsms/invoices/enterprise-for-vendor` (the actual submit) uses `SubmitInvoiceForVendorSchema` which has no `s3Key` field. The uploaded PDF `s3Key` is returned to the browser but cannot be passed to the invoice record — it is orphaned in GCS. Enterprise-entered invoices always have `s3Key = null` regardless of whether a PDF was uploaded.

EA submits invoice:

```
POST /bsms/invoices/enterprise-for-vendor
Body: { vendorRegistryId, invoiceRef, invoiceDate, amountPaise, spendCategory, currency, dueDate? }
```

### Controller → `InvoiceService.submitForVendor`

1. `vendorRegistryRepo.findVerifiedByIdAndEnterprise(vendorRegistryId, enterpriseId)` — validates vendor belongs to this enterprise.

2. Creates invoice via `invoiceRepo.createInvoice` with:
   - `status = AI_VALIDATED` — skips AI entirely, EA provided all fields
   - `source = ENTERPRISE_ENTERED`
   - Real `invoiceRef`, `invoiceDate`, `amountMinorUnit` from the form
   - `s3Key = null` — PDF not linked (see gap above)

3. Writes `SUBMITTED` audit event.

4. Publishes `InvoiceSubmittedEvent`.

**Result:** Invoice lands directly at `AI_VALIDATED` — straight into FE/EA verify queue. No AI processing step.

---

## Journey 3 — Finance Executive verifies

FE sees invoice in their pending queue (status `AI_VALIDATED` or `AI_FLAGGED`). They select a budget node and click **Verify invoice**.

```
POST /bsms/invoices/:id/verify
Body: { orgNodeId }
```

### Controller → `InvoiceService.verifyInvoice`

**File:** `invoice.service.ts`

1. Loads invoice. Guards: must exist, must belong to enterprise, must be `AI_VALIDATED` or `AI_FLAGGED`.

2. Extracts `invoice.amountMinorUnit` as `effectiveAmount` and `invoice.spendCategory` as `category`.

3. Calls `runVerifyLimitCheck(this.limitEngine, orgNodeId, category, effectiveAmount)`.

### `runVerifyLimitCheck` (in `invoice-limit-engine.ts`)

```typescript
const credit = await engine.checkCreditLimit(orgNodeId, amountPaise);       // isReserved defaults to false
if (!credit.passed) return InvoiceStatus.REJECTED;

const spend = await engine.checkSpendLimit(orgNodeId, category, amountPaise);
if (!spend.passed) return InvoiceStatus.SPEND_LIMIT_FLAGGED;

return InvoiceStatus.READY_FOR_PAYMENT;
```

**What each check does:**

`checkCreditLimit(isReserved=false)`:
- Walks OrgNode tree from `orgNodeId` to root
- At COMPANY node: checks `usedAmount + amount <= allocatedAmount` (hard block)
- Uses `engine.check(isReserved=false)` → also writes null-category `VelocityCounter` records as a side effect (benign — no null-category limits exist, these records are never read)
- `isReserved=false` is correct here: invoice not yet reserved, amount must be included in the formula

`checkSpendLimit(isReserved=true)`:
- Uses `engine.check(isReserved=true, categoryKey=category)` → validate-only, no counter writes
- Formula: `usedAmount + pendingAmount <= limitAmount` (existing usage only — does NOT include current invoice amount)
- **Known gap**: for same-day invoices this means the invoice can pass verify even if it would push spend over the category limit. The limit would only catch if existing spend already exceeds the limit. Future-dated invoices are properly caught by `reserveNodeCredit()` → `engine.reserve()` which uses `usedAmount + pendingAmount + amount`. Fix requires architect to add `dryRun: true` mode to engine.

**Results from `runVerifyLimitCheck`:**

| Result | Meaning | Next status |
|---|---|---|
| `REJECTED` | Credit limit exceeded | `REJECTED` |
| `SPEND_LIMIT_FLAGGED` | Credit OK, but category budget already exhausted | `SPEND_LIMIT_FLAGGED` |
| `READY_FOR_PAYMENT` | Both checks passed | `READY_FOR_PAYMENT` |

4. Back in `verifyInvoice`:
   - If `REJECTED`: updates status, writes rejection audit, publishes `InvoiceRejectedEvent`.
   - Otherwise: updates status to `READY_FOR_PAYMENT` or `SPEND_LIMIT_FLAGGED`, writes `VERIFIED` audit, publishes `InvoiceVerifiedEvent`.

5. `orgNodeId` is stored on the invoice — every subsequent step uses it.

---

## Journey 4 — Finance Controller schedules payment

FC sees invoice in their queue (status `READY_FOR_PAYMENT` or `SPEND_LIMIT_FLAGGED`). For `SPEND_LIMIT_FLAGGED`, UI shows a callout explaining a partial payment will be auto-computed. FC sets only the **payment date** and clicks **Approve for payment**.

```
POST /bsms/invoices/:id/approve-for-payment
Body: { paymentDate: "2026-06-10" }
```

### Controller → `InvoiceService.approveForPayment`

**File:** `invoice.service.ts`

1. Loads invoice. Guards: must be `READY_FOR_PAYMENT` or `SPEND_LIMIT_FLAGGED`.

2. **Auto-compute `payment_amt`** (FC never types an amount):

   ```
   READY_FOR_PAYMENT:   payment_amt = invoice_amt  (full payment)
   SPEND_LIMIT_FLAGGED: payment_amt = min(getAvailableSpend(), invoice_amt)
   ```

   For `SPEND_LIMIT_FLAGGED`, calls `limitEngine.getAvailableSpend(orgNodeId, category)` which returns `VelocityCounter.limitAmount - usedAmount - pendingAmount` for this node.

   **Zero-paise guard**: if `availablePaise = 0`, rejects the invoice immediately with `InvoiceStatus.REJECTED` — does not send a 0-paise drawdown to LMS. Writes rejection audit, publishes `InvoiceRejectedEvent`.

3. **Compute `note`**: `payment_amt < invoice_amt` → `"Partial payment: X of Y paise"`. Otherwise `null`.

4. **Routing by date:**

   **Today (`payment_date ≤ today`):**
   - Calls `runPaymentDrawdown(invoice, payment_amt, userId, enterpriseId, note, wasReserved=false)` immediately.
   - No limit re-check — limit was verified at FE verify step.

   **Future date:**
   - Calls `limitEngine.reserveNodeCredit(orgNodeId, category, payment_amt)`:
     - Writes `VelocityCounter.pendingAmount += payment_amt` for all nodes ✓
     - Should also write `OrgNode.pendingAmount += payment_amt` ← **architect gap, not yet implemented**
   - Updates invoice status to `PAYMENT_SCHEDULED`, stores `scheduledPaymentDate` and `scheduledPaymentAmtPaise`.
   - Enqueues delayed BullMQ job on `INVOICE_PAYMENT_QUEUE`:
     ```json
     { "invoiceId": "...", "enterpriseId": "..." }
     ```
     `delay = payment_date.getTime() - Date.now()`.
   - Writes `PAYMENT_SCHEDULED` audit event with `note` and `amountPaise`.
   - Publishes `InvoicePaymentScheduledEvent`.

---

## Journey 5 — BullMQ fires on payment_date

When scheduled job fires, BullMQ worker calls:

```
POST /internal/invoices/:id/process-payment
Body: { enterpriseId }
```

### `InvoiceService.processScheduledPayment`

**File:** `invoice.service.ts`

**Idempotent** — if invoice is no longer `PAYMENT_SCHEDULED`, releases pending reservation (safety net) and writes `PAYMENT_SKIPPED` audit. The safety net fires if the overdue sweep or any other path changed status before the job ran.

1. Loads `payment_amt` from `invoice.scheduledPaymentAmtPaise`. Fallback to `invoice.amountMinorUnit` — data-integrity guard only.

2. **Re-checks BOTH limits** on payment_date:

   **Check A — credit limit (`isReserved=true`):**
   ```typescript
   const credit = await this.limitEngine.checkCreditLimit(orgNodeId, payment_amt, true);
   ```
   Formula: `OrgNode.usedAmount + OrgNode.pendingAmount <= OrgNode.allocatedAmount`

   This invoice's amount is already in `pendingAmount` (placed by `reserveNodeCredit` at FC approval), so the check naturally includes it without double-counting.

   **Current limitation**: `OrgNode.pendingAmount` is never written by `reserve()` (architect gap), so this check always reads `0 + 0`. Will work correctly once architect fixes `reserve()`.

   If fails → `limitEngine.releaseNodeCredit(orgNodeId, category, payment_amt)` → then `_rejectScheduledPayment`.

   **Check B — spend limit (`isReserved=true`):**
   ```typescript
   const spend = await this.limitEngine.checkSpendLimit(orgNodeId, category, payment_amt);
   ```
   Formula: `VelocityCounter.usedAmount + VelocityCounter.pendingAmount <= limitAmount`

   VelocityCounter is correctly maintained (no architect gap here) — catches the case where category budget was tightened between FE verify and payment_date.

   If fails → same `releaseNodeCredit` + `_rejectScheduledPayment` path.

3. Both pass → calls `runPaymentDrawdown(invoice, payment_amt, null, enterpriseId, null, wasReserved=true)`.

4. Writes `PAYMENT_PROCESSED` audit event.

---

## Journey 6 — `runPaymentDrawdown` (the actual payment)

**File:** `invoice.service.ts` — private method, called from:
- `approveInvoice` (FM/CFO direct path, no prior reserve → `wasReserved=false`)
- `approveForPayment` when `payment_date = today` → `wasReserved=false`
- `processScheduledPayment` when BullMQ fires → `wasReserved=true`

### What it does

1. **LMS drawdown** — calls `lmsGateway.createDrawdown(drawdownInput)`:
   ```typescript
   { enterpriseId, invoiceId, invoiceRef, amountPaise: payment_amt, currency }
   ```
   LMS = Frappe Lending. Creates actual drawdown against enterprise's LOC. Returns `{ drawdownRef }`.

2. **Settle limits** (only if `invoice.orgNodeId && invoice.spendCategory`):
   ```typescript
   await this.limitEngine.settleNodeSpend(orgNodeId, spendCategory, payment_amt, wasReserved);
   ```
   - `wasReserved=true`: `engine.commit(isReserved=true)` → `VelocityCounter`: `commitPending` (pending→used). Should also update `OrgNode.usedAmount += amount, OrgNode.pendingAmount -= amount` ← **architect gap**.
   - `wasReserved=false`: `engine.commit(isReserved=false)` → `VelocityCounter`: `increment`. Should also update `OrgNode.usedAmount += amount` ← **architect gap**.

3. **Final status:**
   ```typescript
   const isPartial   = payment_amt < invoice.amountInr;
   const finalStatus = isPartial ? InvoiceStatus.PARTIALLY_PAID : InvoiceStatus.APPROVED;
   ```

4. **Updates invoice** to `APPROVED` or `PARTIALLY_PAID`, stores `lmsDrawdownRef`, `approvedAmountPaise`, `reviewedAt`, `reviewedByUserId` (null when called from worker).

5. **Writes `APPROVED` audit event** with `note` (non-null when partial) and `amountPaise`.

6. **Publishes `InvoiceApprovedEvent`** with `drawdownRef` and `payment_amt`.

---

## Reject paths

### Manual reject (FM/CFO/EA/FE)

```
POST /bsms/invoices/:id/reject
Body: { rejectionReason }
```

→ `InvoiceService.rejectInvoice`

Guard: cannot reject if status in `TERMINAL_STATUSES`:
- `READY_FOR_PAYMENT` / `SPEND_LIMIT_FLAGGED` — verified, must proceed to FC
- `PAYMENT_SCHEDULED` — BullMQ job exists; rejecting would orphan it
- `APPROVED`, `PARTIALLY_PAID`, `REJECTED`, `CANCELLED`, `CONVERTED`, `PAID` — already terminal

Writes `REJECTED` status, audit, publishes `InvoiceRejectedEvent`.

### Scheduled payment rejected at execution

Inside `processScheduledPayment` → `_rejectScheduledPayment`. Always calls `releaseNodeCredit` first to undo the pending reservation before rejecting.

---

## Journey 7 — Dispute flow

### Vendor raises dispute

Vendor can dispute after:
- Invoice is `REJECTED`, OR
- Invoice is `APPROVED` / `CONVERTED` / `PAID` with `approvedAmountPaise < invoice.amountInr` (partial payment)

```
POST /bsms/invoices/:id/dispute
Body: { reason }
```

→ `InvoiceDisputeService.raiseDispute`

1. Loads invoice. Verifies vendor owns the registry via `vendorRegistryRepo.findByIdAndUserId`.
2. Validates disputable condition (rejected OR approved-partial).
3. Blocks if open dispute already exists (`OPEN` or `UNDER_REVIEW`).
4. `disputeRepo.upsertForRaise` — creates/reopens dispute record.
5. Writes `DISPUTE_RAISED` audit event with `actorRole = 'VENDOR'`.
6. Publishes `InvoiceDisputeRaisedEvent`.

### FM/CFO resolves dispute

```
POST /bsms/invoices/:id/dispute/resolve
Body: { action: 'ACCEPT' | 'DISMISS', resolutionNote, newApprovedAmountPaise? }
```

→ `InvoiceDisputeService.resolveDispute`

`ACCEPT` outcomes:
- Invoice was `REJECTED` → re-opens invoice to `AI_VALIDATED` (back to verify queue)
- Invoice was approved-partial + `newApprovedAmountPaise` provided → adjusts `approvedAmountPaise`
- Amount validation: must be > 0 and ≤ original `invoice.amountInr`

`DISMISS` → closes dispute with no invoice change.

All changes via `disputeRepo.resolveAtomically` — dispute update + invoice update in a single transaction.

Writes `DISPUTE_RESOLVED` or `DISPUTE_DISMISSED` audit. Publishes `InvoiceDisputeResolvedEvent`.

---

## Known platform gaps (architect fixes pending)

These are in `packages/platform/src/engines/spend-limit/spend-limit.engine.ts` — not our code to fix.

| Gap | Impact | Fix needed |
|---|---|---|
| `reserve()` doesn't write `OrgNode.pendingAmount` | `checkCreditLimit(isReserved=true)` at payment_date reads 0 — credit check ignores all reservations | `reserve()` must `OrgNode.pendingAmount += amount` for all path nodes |
| `commit()` doesn't write `OrgNode.usedAmount` | After every payment, `OrgNode.usedAmount` stays 0 — credit baseline always stale | `commit()` must `OrgNode.usedAmount += amount` (and `pendingAmount -= amount` when `isReserved=true`) |
| `releasePending()` doesn't decrement `OrgNode.pendingAmount` | Stale pending on cancellation | `releasePending()` must `OrgNode.pendingAmount -= amount` |
| `checkSpendLimit(isReserved=true)` at verify skips current invoice amount | Same-day invoices that would push over category limit pass verify as READY_FOR_PAYMENT | Architect needs `dryRun: true` mode: uses `usedAmount + amount` formula without writing counters |
| `checkCreditLimit(isReserved=false)` writes null-category VelocityCounters | DB noise — records never read, no functional impact | Architect to add `checkCreditOnly()` that reads OrgNode directly, skips VelocityCounter entirely |

**What works correctly today:**

- Category limits (`VelocityCounter`) are fully maintained — `reserve()`, `commit()`, `releasePending()` all write correctly
- Credit limit catches single-invoice overruns (one invoice > full credit limit → REJECTED at verify)
- Zero-paise drawdown guard prevents 0-paise LMS requests when budget is exhausted
- Dispute flow complete end-to-end

**What is broken until architect fix:**

- Two invoices of ₹1500 each against ₹2000 credit limit: both APPROVE (usedAmount always 0)
- Scheduled payment credit re-check on payment_date always passes (pendingAmount always 0 on OrgNode)

**What changes after architect fix:**

- `checkCreditLimit` at payment_date already passes `isReserved=true` (line 323 in `invoice.service.ts`) — will be correct immediately after fix
- `checkCreditLimit` at verify already uses `isReserved=false` (default) — correct formula, will work once `OrgNode.usedAmount` is maintained

---

## Supporting Services

### `InvoiceStorageService` (`invoice-storage.service.ts`)

Wraps Google Cloud Storage.

- `getSignedUploadUrl(enterpriseId, filename, contentType)` — returns write-signed URL (15 min) + GCS path `invoices/{enterpriseId}/{uuid}/{filename}`. Vendor browser uploads directly to GCS.
- `getSignedReadUrl(gcsPath, expiresIn)` — returns read-signed URL. FM/CFO/FE "View PDF" button uses 900s (15 min). Vendor PDF view also 900s. Returns `null` in stub mode.
- `save(buffer, enterpriseId, filename, mimeType)` — direct server-side upload (used by `uploadInvoicePdf` multipart path).

### `InvoiceSseService` (`invoice-sse.service.ts`)

Subscribes to `eventBus` with wildcard `*` on boot. Filters `eventType.startsWith('invoice.')`. Emits to `EventEmitter` keyed by `enterpriseId`.

Frontend keeps persistent `GET /bsms/invoices/events`. Any invoice event for that enterprise delivers `{ type, invoiceId }` → frontend re-fetches list. No polling needed.

Heartbeat: empty `data: ''` frame every 25s — prevents proxy/load-balancer idle timeout (Nginx default 60s).

### `InvoiceAuditRepository` (`invoice-audit.repository.ts`)

Append-only. Every state transition writes to `InvoiceAuditEvent`:
- `action` — `InvoiceAuditAction` enum
- `actorName` / `actorRole` — snapshot at action time (never a join)
- `note` — rejection reasons, partial amounts, skip reasons
- `amountPaise` — the amount involved (null for verify/reject without amount)

`resolveUser(userId)` — best-effort User lookup for audit snapshot. Never throws — returns `{ name: null, role: null }` on error.

### `IInvoiceLimitEngine` (`invoice-limit-engine.ts`)

Interface with injection token `INVOICE_LIMIT_ENGINE`.

**Three implementations:**

| Class | Used in | Behaviour |
|---|---|---|
| `InvoiceLimitEngineStub` | `bsms.module.ts` (current default) | All checks pass, all mutations no-ops, `getAvailableSpend` returns MAX_SAFE_INTEGER |
| `RealInvoiceLimitEngine` | Switch in `bsms.module.ts` when ready | Wraps `SpendLimitEngine` from `packages/platform` |

To enable real implementation:
```typescript
// bsms.module.ts providers:
{ provide: INVOICE_LIMIT_ENGINE, useClass: RealInvoiceLimitEngine }
```

**`checkCreditLimit` signature:**
```typescript
checkCreditLimit(orgNodeId: string, amountPaise: bigint, isReserved?: boolean): Promise<{ passed: boolean }>
```
- `isReserved=false` (default) — used at FE verify: `usedAmount + amount <= allocatedAmount`
- `isReserved=true` — used at payment_date re-check: `usedAmount + pendingAmount <= allocatedAmount`

### `BullMqShutdownService` (`bull-mq-shutdown.service.ts`)

Listens for `SIGTERM`. Drains `INVOICE_ANALYSIS_QUEUE` and `INVOICE_PAYMENT_QUEUE`, closes Redis connection. Prevents mid-flight jobs on process exit.

---

## Domain Events (`bsms.events.ts`)

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

All events carry `enterpriseId` and `invoiceId`.

---

## Invoice Status State Machine

```
SUBMITTED
  └─ AI_PROCESSING  (atomicSetAiProcessing — DB + audit in one transaction)
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
  ├─ REJECTED            (approveForPayment → payment_amt=0 guard)
  └─ PAYMENT_SCHEDULED   (approveForPayment → future date)
  (rejectInvoice BLOCKED — both statuses are in TERMINAL_STATUSES)

PAYMENT_SCHEDULED
  ├─ REJECTED            (processScheduledPayment → limit check fail OR overdue sweep)
  ├─ APPROVED            (processScheduledPayment → drawdown → full)
  └─ PARTIALLY_PAID      (processScheduledPayment → drawdown → partial)

APPROVED / PARTIALLY_PAID
  └─ dispute: ACCEPT → AI_VALIDATED (re-opened) or adjusted amount (stays APPROVED/PARTIALLY_PAID)

APPROVED / PARTIALLY_PAID / REJECTED → terminal (no further transitions except dispute)
```

`TERMINAL_STATUSES` blocks any action on these statuses. `PAYMENT_SCHEDULED` is also in the set — once a BullMQ job exists, manual rejection is blocked to prevent orphaned jobs.

---

## All amounts are paise as BigInt

Every amount (DB columns, service variables, event fields) is stored and computed as `BigInt` paise. `invoice_amt` and `payment_amt` in the flow diagram map directly to `invoice.amountInr` and `payment_amt` BigInt variables in the service. Never float, never Decimal, never rupees inside the service layer.
