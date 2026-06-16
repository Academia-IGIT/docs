# Logging & Observability — full walkthrough (NEXO-168)

How every added/changed file fits together: what it is, what it exports, where it's
used, and how it's called. All paths relative to repo root.

---

## 0. Mental model

One shared package, **`packages/observability`** (`@nexo/observability`), owns all
provider-specific telemetry. Three concerns:

- **Logging** → Pino (stdout). Backend only (api, worker).
- **Tracing** → OpenTelemetry. Backend; exported to Sentry via OTLP.
- **Errors + logs → Sentry** → a thin provider-agnostic facade (Sentry today, swappable).

App code imports **only** `@nexo/observability`. It never imports `pino`,
`@sentry/*`, or `@opentelemetry/*` directly. (Web is the exception: it uses
`@sentry/nextjs` directly because browser features — replay, browser tracing — are
best left to the SDK.)

Boot order matters: each backend app imports an `observability/instrument` module
**first**, which loads `.env`, then starts OTel, monitoring, and the logger before
anything else loads.

---

## 1. The shared package — `packages/observability/src/`

### `index.ts`
Barrel — the public API. Everything app code can import. (`sampler.ts` is NOT
exported — it's internal to `telemetry.ts`.)

### `logger.ts` — Pino logging
- **Exports:** `createLogger(opts)`, `getLogger()`, types `Logger`, `LoggerOptions`.
- **Purpose:** build the process-wide Pino logger. JSON in prod / pretty in dev;
  bakes `service.name` + `environment`; redacts secrets/PII; serializes Errors on
  `err`/`error`/`reason`; a `mixin` injects correlation fields (requestId, traceId,
  spanId, userId, tenantId) on every line by calling `resolveContext()` from
  `context.ts`.
- **Called by:**
  - `createLogger` — once per process in each `observability/instrument.ts`
    (`apps/api`, `apps/worker`).
  - `getLogger` — everywhere logs are emitted: `apps/api/src/common/exception.filter.ts`,
    `apps/api/src/observability/http-logging.interceptor.ts`,
    `apps/api/src/observability/pino-nest-logger.ts`, worker jobs
    (`apps/worker/src/jobs/*.job.ts`), `packages/platform/src/events/event-bus.ts`,
    `packages/platform/src/gateways/notifications/{sms,email}.gateway.ts`,
    `apps/api/src/modules/kyb/kyb.repository.ts`,
    `apps/api/src/modules/bsms/webhooks/handlers/clearing.handler.ts`.

### `context.ts` — request/job correlation context
- **Exports:** `setContextResolver(fn)`, `resolveContext()`, `runWithContext(ctx, fn)`,
  `patchContext(partial)`, `traceFields()`, type `ObsContext`.
- **Purpose:** supply the per-request/per-job fields the logger mixes in.
  - `traceFields()` reads traceId/spanId from the active OTel span.
  - The host registers a resolver via `setContextResolver` for requestId/userId/tenantId.
  - `runWithContext` is an AsyncLocalStorage used by the worker (no nestjs-cls there).
- **Called by:**
  - `resolveContext` — by the logger `mixin` in `logger.ts` (every log line) and by
    `providers/sentry.ts` (error tags).
  - `setContextResolver` — by `apps/api/src/observability/observability.module.ts`
    (wires it to nestjs-cls). Worker leaves the default (reads `runWithContext`).
  - `runWithContext` — by `instrumentProcessor` in `bullmq.ts`.

### `telemetry.ts` — OpenTelemetry
- **Exports:** `initTelemetry(opts)`, `shutdownTelemetry()`, `withSpan(name, fn, attrs?)`,
  `injectTraceContext()`, `withExtractedContext(carrier, fn)`, types `TelemetryOptions`,
  `TraceCarrier`.
- **Purpose:**
  - `initTelemetry` — starts the `NodeSDK`: explicit instrumentations
    (http/undici/express/ioredis/pg), the sampler from `sampler.ts`, and an
    OTLP trace exporter (to Sentry) when `OTEL_EXPORTER_OTLP_ENDPOINT` is set.
    `parseOtlpHeaders` parses `OTEL_EXPORTER_OTLP_HEADERS` (Sentry's `x-sentry-auth`).
  - `withSpan` — run a function inside a span (records exceptions, sets status).
  - `injectTraceContext` / `withExtractedContext` — W3C trace carrier in/out for
    crossing the BullMQ boundary.
- **Called by:**
  - `initTelemetry` / `shutdownTelemetry` — `observability/instrument.ts` (both apps);
    `shutdownTelemetry` also in `apps/api/src/main.ts` + `apps/worker/src/main.ts` SIGTERM.
  - `withSpan` / `injectTraceContext` / `withExtractedContext` — inside `bullmq.ts`
    and `packages/platform/src/events/bullmq-event-bus.ts`.

### `sampler.ts` — trace sampling (internal)
- **Exports:** `createMoneyAwareSampler(ratio?)`, `__test` (for unit tests). Not in `index.ts`.
- **Purpose:** decide which traces to keep. `ParentBased` (whole-trace integrity) +
  money-aware root sampler — money-critical URL paths always sampled, rest at
  `OTEL_TRACES_SAMPLE_RATIO` (default 0.1).
- **Called by:** `telemetry.ts` only — `new NodeSDK({ sampler: createMoneyAwareSampler() })`.
- **NOTE:** sampling strategy is still under review (see §8).

### `monitoring.ts` — provider-agnostic error/log facade
- **Exports:** `getMonitoring()`, `setMonitoringProvider(p)`, `createMonitoring(opts)`,
  types `MonitoringProvider`, `MonitoringContext`, `MonitoringUser`, `Breadcrumb`,
  `MonitoringInitOptions`.
- **Purpose:** the only API app code uses for error reporting
  (`reportError`/`reportMessage`/`setUser`/`addBreadcrumb`/`flush`). `createMonitoring`
  picks the provider: Sentry when `SENTRY_DSN` set, else no-op; **throws in production**
  if no DSN (unless `OBSERVABILITY_ALLOW_NOOP`); honors the `OBSERVABILITY_DISABLED`
  kill switch.
- **Called by:**
  - `createMonitoring` — `observability/instrument.ts` (both apps).
  - `getMonitoring` — `apps/api/src/common/exception.filter.ts`, `apps/api/src/main.ts`
    (SIGTERM flush), `apps/worker/src/main.ts`, `bullmq.ts` (job failures),
    `process-handlers.ts`.

### `providers/sentry.ts` — the Sentry implementation
- **Exports:** `SentryMonitoringProvider`, `SentryProviderOptions`.
- **Purpose:** the ONLY backend file importing `@sentry/node`. `Sentry.init` with:
  `skipOpenTelemetrySetup: true` (OTel owns tracing), disabled crash integrations
  (we own those), `enableLogs: true` + `pinoIntegration` (forward pino logs to Sentry
  Logs), `beforeSendLog` → `scrubAttributes` (PII scrub). Tags events with the active
  traceId.
- **Called by:** `monitoring.ts` `createMonitoring` (instantiates it).

### `providers/noop.ts` — no-op provider
- **Exports:** `NoopMonitoringProvider`.
- **Purpose:** silent provider when no DSN / observability disabled. The point of the
  swappable design.
- **Called by:** `monitoring.ts`.

### `bullmq.ts` — BullMQ trace correlation
- **Exports:** `withTraceData(data)`, `instrumentProcessor(queueName, processor)`.
- **Purpose:**
  - `withTraceData` — producer side: attaches the W3C trace carrier (`__trace`) to job
    payload so the consumer joins the same trace.
  - `instrumentProcessor` — consumer side: wraps a processor → extracts trace context,
    opens a `job <queue>` span, runs in job context (logs carry queueName/jobName/jobId/
    tenantId), reports failures once.
- **Called by:**
  - `withTraceData` — `apps/api/src/modules/bsms/invoices/invoice.service.ts`,
    `apps/api/src/modules/bsms/reimbursements/reimbursement-flow.service.ts`,
    `apps/api/src/modules/ocr/ocr-submit.service.ts`,
    `packages/platform/src/events/bullmq-event-bus.ts` (`publish`).
  - `instrumentProcessor` — `apps/worker/src/main.ts` wraps every `new Worker(...)`.

### `process-handlers.ts` — crash net
- **Exports:** `installProcessHandlers(opts?)`, `ProcessHandlerOptions`.
- **Purpose:** `uncaughtException` / `unhandledRejection` → log fatal + report +
  flush + exit (so the orchestrator restarts clean).
- **Called by:** `apps/api/src/observability/instrument.ts` and
  `apps/worker/src/main.ts`.

### `log-scrub.ts` — Sentry-log PII scrubber
- **Exports:** `scrubAttributes(attrs)`, `isSensitiveKey(key)`.
- **Purpose:** `pinoIntegration` captures raw log args BEFORE pino's redaction, so this
  scrubs sensitive values by key name in `beforeSendLog` before they leave the process.
- **Called by:** `providers/sentry.ts` `beforeSendLog`. (Web has its own copy — see §4.)

### Tests — `*.spec.ts`
`logger.spec.ts`, `context.spec.ts`, `monitoring.spec.ts`, `bullmq.spec.ts`,
`sampler.spec.ts`, `log-scrub.spec.ts` — 41 tests. Config: `jest.config.js`,
`tsconfig.json` (IDE/typecheck, incl specs), `tsconfig.build.json` (emit, excludes specs),
`eslint.config.mjs`.

---

## 2. API wiring — `apps/api/`

### `src/observability/instrument.ts`  *(first import in main.ts)*
Loads `.env` (walks up from file — runs before Nest's ConfigModule), then
`initTelemetry`, `createMonitoring`, `createLogger`, `installProcessHandlers` — all for
`service: 'api'`.

### `src/main.ts`
- Line 3: `import './observability/instrument';` — must be first.
- Attaches `PinoNestLogger` via `app.useLogger(...)` (+ `bufferLogs`).
- Registers `HttpLoggingInterceptor` globally.
- SIGTERM → `getMonitoring().flush()` + `shutdownTelemetry()`.

### `src/observability/pino-nest-logger.ts`
- **Purpose:** `LoggerService` adapter so every NestJS `new Logger(...)` (and framework
  logs) route through Pino. Wraps `getLogger()`.
- **Called by:** `main.ts` `app.useLogger(new PinoNestLogger())`.

### `src/observability/http-logging.interceptor.ts`
- **Purpose:** one structured access-log line per request (method, route pattern with
  params/query stripped, status, durationMs). Skips `/health`, `/readyz`, `/docs`.
- **Called by:** `main.ts` `app.useGlobalInterceptors(...)`.

### `src/observability/observability.module.ts`
- **Purpose:** on init, `setContextResolver(...)` wired to nestjs-cls — so requestId/
  userId/tenantId from the active request flow into every log line.
- **Called by:** imported in `src/app.module.ts`.

### `src/common/exception.filter.ts` (modified)
- Uses `getLogger()` (structured) + `getMonitoring().reportError()` for unexpected 5xx —
  the **single** in-request error report path. 4xx logged, not reported.

### `src/modules/health/` (new)
- `health.controller.ts` — `GET /health` (liveness), `GET /readyz` (readiness). `@Public()`.
- `health.service.ts` — runs DB + Redis checks; returns 503 if degraded.
- `health.repository.ts` — DB ping (`$queryRawUnsafe('SELECT 1')`); raw DB lives in a
  repository per the fitness rule.
- `health.module.ts` — wires the three. Registered in `app.module.ts`.

### Trace producers (modified — added `withTraceData`)
- `src/modules/bsms/invoices/invoice.service.ts` — `paymentQueue.add('process-payment', withTraceData({...}))`.
- `src/modules/bsms/reimbursements/reimbursement-flow.service.ts` — same for reimbursement payment.
- `src/modules/ocr/ocr-submit.service.ts` — OCR input job.

### Console → logger (modified)
- `src/modules/kyb/kyb.repository.ts`, `src/modules/bsms/webhooks/handlers/clearing.handler.ts`.

---

## 3. Worker wiring — `apps/worker/`

### `src/observability/instrument.ts`  *(first import)*
Loads `.env`, then `initTelemetry`/`createMonitoring`/`createLogger` for `service: 'worker'`.

### `src/main.ts` (modified)
- Line 3: `import './observability/instrument';`.
- Wraps **every** `new Worker(...)` with `instrumentProcessor(queueName, processor)`.
- `installProcessHandlers()` for crashes.
- `w.on('error', ...)` → `getLogger()` (worker-level events only; per-job logs come from
  `instrumentProcessor`).
- SIGTERM → `shutdownTelemetry()`.

### `src/jobs/*.job.ts` (modified)
`console.*` → `getLogger()` with structured fields (no payloads/PII):
`invoice-payment.job.ts`, `invoice-payment-sweep.job.ts`, `invoice-overdue-sweep.job.ts`,
`reimbursement-payment.job.ts`, `reimbursement-payment-sweep.job.ts`.

---

## 4. Web wiring — `apps/web/`

Uses `@sentry/nextjs` directly (browser SDK features). Does NOT import `@nexo/observability`
(would pull Node deps into the browser bundle).

### `src/instrumentation-client.ts` (new)
Browser `Sentry.init`: errors, breadcrumbs, `browserTracingIntegration`,
`replayIntegration`, `consoleLoggingIntegration` (logs), `enableLogs`,
`tracePropagationTargets`, `beforeSendLog` → `scrubLogAttributes`, master kill switch
(`NEXT_PUBLIC_OBSERVABILITY_DISABLED`). Exports `onRouterTransitionStart`.

### `src/sentry.server.config.ts` + `src/sentry.edge.config.ts` (new)
`Sentry.init` for the Next server / edge runtimes (errors + logs + scrub + kill switch).

### `src/instrumentation.ts` (new)
Next's `register()` — loads the right server/edge config per runtime. Exports
`onRequestError = Sentry.captureRequestError`.

### `src/lib/observability/trace.ts` (new)
- `traceHeaders()` — returns `sentry-trace` + `baggage` + a derived W3C `traceparent`
  for outbound API calls (so web→api joins one trace; backend OTel reads `traceparent`).
- **Called by:** `src/lib/api/client.ts` `apiFetch` — spread into request headers.

### `src/lib/observability/scrub.ts` (new)
Browser-safe copy of the log scrubber (mirrors `packages/observability/src/log-scrub.ts`).
Used by all three web Sentry configs' `beforeSendLog`.

### `next.config.js` (modified)
Wrapped with `withSentryConfig` — source-map upload at `next build` (org/project/auth-token).

---

## 5. Platform wiring — `packages/platform/src/`

- `events/bullmq-event-bus.ts` — `publish()` wraps event with `withTraceData`; the worker
  side wraps `dispatch` in `withExtractedContext` + `withSpan` (domain events join the trace).
- `events/event-bus.ts` — in-memory bus error log → `getLogger()`.
- `gateways/notifications/sms.gateway.ts` + `email.gateway.ts` — stub OTP/invite logs → `getLogger()`.
- `auth/auth.module.ts` — removed a duplicate `ClsModule.forRoot` (CLS registered once by
  `platform.module.ts`; double-mount regenerated requestId).

---

## 6. End-to-end flows

**Log line:** `getLogger().info({...}, 'msg')` → Pino → `mixin` calls `resolveContext()`
(context.ts) → adds requestId (cls/ALS) + traceId/spanId (active OTel span) → stdout (JSON
prod / pretty dev). If level ≥ `SENTRY_LOG_LEVELS`, `pinoIntegration` forwards a copy to
Sentry Logs, scrubbed by `beforeSendLog`.

**Error (API):** thrown → `AppExceptionFilter` → `getLogger().error` + `getMonitoring().reportError`
→ Sentry Issues (tagged service.name + traceId), sanitized 500 to client.

**Trace:** browser span → `apiFetch` sends `traceparent` → API http instrumentation continues
the trace → producer `withTraceData` puts the carrier on the job → worker `instrumentProcessor`
extracts it + spans the job → all spans exported to Sentry OTLP under one traceId.

---

## 7. Config / deploy files

- `.env` / `.env.example` — observability env (commented defaults; `OBSERVABILITY_DISABLED`).
- `apps/web/.env.local` / `apps/web/.env.example` — web Sentry env.
- `infra/k8s/base/api-deployment.yaml` + `worker-deployment.yaml` — `SENTRY_DSN` (secret),
  `OTEL_EXPORTER_OTLP_*`, `SENTRY_TRACES_SAMPLE_RATE`.
- `apps/web/Dockerfile` — `ARG`/`ENV` Sentry build-args before `next build`.
- `cloudbuild.yaml` — web build-args (DSN/org/project substitutions + `SENTRY_AUTH_TOKEN`
  secret), `availableSecrets`.
- Deploy prerequisites: see `sentry-handoff.md` in this folder.

---

## 7b. Environment variables — exact behavior

All optional unless noted. **Backend vars** (`api`/`worker`) live in repo-root `.env`.
**Web vars** (`NEXT_PUBLIC_*` + source-map vars) live in `apps/web/.env.local` — the
root `.env` does NOT feed the web app.

### Backend (root `.env`)
| Var | Default | What it does |
|---|---|---|
| `OBSERVABILITY_DISABLED` | `false` | Master kill switch. `true` → no Sentry init (no errors, no logs forwarded) **and** no trace export. stdout logging still works. |
| `LOG_LEVEL` | `info` | Minimum stdout log level (a **floor**). See gotcha below. |
| `LOG_PRETTY` | on in dev / off in prod | Output **format** toggle only. `true` → colored pretty; `false` → raw JSON. Not a level. |
| `SENTRY_DSN` | _(empty)_ | Backend error/log destination. Empty → no-op provider in dev; **throws in production** (fail-fast). |
| `SENTRY_LOG_LEVELS` | `warn,error,fatal` | Which log levels are **forwarded to Sentry Logs** (an **exact set**). See gotcha below. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | _(empty)_ | Where traces export. Empty → traces collected locally but not sent. Set to Sentry's OTLP URL to export. |
| `OTEL_EXPORTER_OTLP_HEADERS` | _(empty)_ | Auth header for the OTLP endpoint (`x-sentry-auth=sentry sentry_key=<public-key>`). |
| `OTEL_TRACES_SAMPLE_RATIO` | `0.1` | Fraction (0–1) of non-money traces kept. Money paths always kept. *(sampling strategy under review — §8)* |
| `SENTRY_TRACES_SAMPLE_RATE` | `0.1` | **No real effect on backend** — Sentry makes no backend transactions (OTel does). Vestigial; ignore. |
| `SENTRY_ORG` / `SENTRY_PROJECT` / `SENTRY_AUTH_TOKEN` | _(empty)_ | Source-map upload at build time only. Token is a **secret**. |

### Web (`apps/web/.env.local`)
| Var | Default | What it does |
|---|---|---|
| `NEXT_PUBLIC_OBSERVABILITY_DISABLED` | `false` | Web master kill switch. `true` → browser/SSR send nothing to Sentry. |
| `NEXT_PUBLIC_SENTRY_DSN` | _(empty)_ | Browser/SSR Sentry destination. Empty → web Sentry effectively off. |
| `NEXT_PUBLIC_SENTRY_ENV` | `NODE_ENV` | Environment label on web events. |
| `NEXT_PUBLIC_SENTRY_TRACES_SAMPLE_RATE` | `0.1` | Fraction of browser transactions traced. **Baked at `next build`** — runtime change has no effect on the browser bundle. |
| `NEXT_PUBLIC_SENTRY_REPLAYS_SESSION_SAMPLE_RATE` | `0.1` | Fraction of sessions replayed. Sessions with errors are always replayed (hardcoded 1.0). |

### Gotchas — the three "log" vars are NOT the same thing
1. **`LOG_LEVEL` is a FLOOR (≥).** It's one value; everything at that level **and above** is emitted.
   `LOG_LEVEL=warn` → emits `warn`, `error`, `fatal` (drops `info`, `debug`, `trace`).
   Order: `trace < debug < info < warn < error < fatal`.
2. **`SENTRY_LOG_LEVELS` is an EXACT SET (membership).** Comma-separated list of the
   *specific* levels to forward to Sentry — **not** a floor.
   `SENTRY_LOG_LEVELS=warn` → forwards **only** `warn`. `error` and `fatal` are **NOT** forwarded.
   To forward warnings + errors you must list them all: `SENTRY_LOG_LEVELS=warn,error,fatal`.
   *(This is the key confusion: `warn` here ≠ "warn and up".)*
3. **`LOG_PRETTY` is format, not level.** Independent axis — controls JSON vs pretty, nothing to do with which logs appear.

**So "do I write only `warn` or all of them?"**
- For `LOG_LEVEL`: write **one** value (the floor). `warn` is enough to also get error/fatal.
- For `SENTRY_LOG_LEVELS`: write **every** level you want forwarded. `warn` alone gives you *only* warn — list `warn,error,fatal` (or `info,warn,error,fatal`) explicitly.

### Relationship
A log must first pass `LOG_LEVEL` (to be emitted at all). Of those emitted, the subset
whose level ∈ `SENTRY_LOG_LEVELS` is *also* copied to Sentry Logs. Example: `LOG_LEVEL=info`
+ `SENTRY_LOG_LEVELS=warn,error,fatal` → info/warn/error/fatal all hit stdout; only
warn/error/fatal also go to Sentry. (If `LOG_LEVEL=error`, a `warn` is never emitted, so
`SENTRY_LOG_LEVELS` listing `warn` does nothing — the floor wins.)

---

## 8. Known open items
- **Trace sampling strategy** under review (`sampler.ts`) — head sampling can't be driven by
  a code-set attribute on inbound HTTP (decision is at request start). Options: 100% now, or
  OTel Collector + tail sampling later.
- Python services (`scoring`, `ocr-agent`) not OTel-instrumented.
- OCR `ocr.input → ocr.output` round-trip trace correlation.
- Deep nested-PII redaction in stdout logs is shallow (Sentry log path scrubbed by key name).
