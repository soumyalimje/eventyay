PR responses and AI review notes

This document collects targeted responses to expected AI/code-review comments and maps them to the code changes in this PR. Use these snippets in the PR conversation.

1) "Why add an error reporter instead of Sentry?"
- Short answer: This PR adds a minimal, low-risk reporter to get immediate telemetry and to avoid blank-screen crashes. It provides best-effort reporting, deduping, and URL-scrubbing. Production-grade monitoring (Sentry/Datadog) is recommended as a follow-up and can be integrated behind a config flag.
- Where: `src/lib/errorReporter.js`.
- Follow-up action: I can add Sentry integration in a follow-up PR (behind `config.sentryDsn`) with source-map upload instructions.

2) "Won't we double-report errors (boundary + global handler)?"
- We implemented in-memory deduping (30s window) in `errorReporter.js` to reduce duplicates for identical errors.
- We also added `config.frontendErrorReporting = false` support to disable reporting in environments where server-side dedupe happens or to avoid duplication during testing.
- Files: `src/lib/errorReporter.js` (dedupe + config flag), `src/main.js` (global handler), `src/components/ErrorBoundary.vue` (calls reporter).

3) "Does this leak PII or tokens?"
- The reporter scrubs query parameters from `window.location.href` before sending to the backend to reduce risk of leaking tokens.
- The reporter payload is intentionally minimal (message, stack, URL without query, userAgent, meta).
- Recommendation: add server-side scrubbing and a privacy policy for collected frontend telemetry.
- File: `src/lib/errorReporter.js` (scrubUrl implementation).

4) "How do we test this locally?"
- See `SMOKE_TEST.md` in `app/eventyay/webapp/SMOKE_TEST.md`.
- Short steps: run dev server, then in browser console trigger `throw new Error('SMOKE')` or `Promise.reject(new Error('SMOKE'))`, observe fallback UI and reporter attempt to POST (or console log if `config.api.feedback` not set).

5) "What about errors outside Vue lifecycle (service worker, socket messages)?"
- We added global handlers for `window.onerror` and `unhandledrejection` in `src/main.js`.
- We added service worker `error` and `messageerror` listeners in `src/main.js` to report SW issues.
- We hardened WebSocket message processing and error events in `src/lib/WebSocketClient.js` and `src/lib/api.js` to report JSON parse errors and dispatch exceptions.

6) "Performance or memory concerns from dedupe map?"
- Deduping uses an in-memory Map and prunes when growing beyond 5000 entries. This keeps memory bounded and cheap. The 30s window should be sufficient to avoid spamming the server while not keeping old entries indefinitely.
- File: `src/lib/errorReporter.js`.

7) "How can we avoid duplicate UI fallback when multiple boundaries exist?"
- Boundaries render a localized fallback for their subtrees. For a full-app fallback, the app-level `ErrorBoundary` is used. For isolated components (media, sidebar), component-level boundaries are present to keep the rest of the page functional.
- If a global full-app fallback is desired, we can modify top-level boundary behavior to swallow errors or show a modal. This was intentionally left as per-component fallback for better availability.

8) "How will this behave in production?"
- If `config.api.feedback` is configured in the injected frontend config, `errorReporter` will POST events to that endpoint; otherwise it falls back to console logs.
- For production, I recommend integrating Sentry (DSN) and enabling source-map uploads during build.

9) "Can we toggle reporting per environment?" 
- Yes. Set `frontendErrorReporting` to `false` in the injected `config` object in environments where you want to disable the reporter (e.g., tests or local dev).
- Example (index.html injection):
  <script>window.eventyay = { ... , frontendErrorReporting: false }</script>

10) "Screenshots / Visuals"
- Included a placeholder screenshot: `app/eventyay/webapp/static/pr_screenshots/error_boundary_fallback.svg`. Replace with a real screenshot from your environment if desired (copy into this path and commit).

11) "Next steps / follow-ups"
- Add Sentry integration behind config flag (I can prepare this PR too).
- Add automated unit tests asserting ErrorBoundary fallback renders when a child throws (Vitest + Vue Test Utils).
- Add server-side collector endpoint or map to an existing feedback endpoint to receive and dedupe events.

---
How I recommend replying to reviewers (copy/paste ready)

- Duplicate reporting: "We've added client-side deduping (30s) and a config flag `frontendErrorReporting` to disable the reporter if server-side dedupe is preferred. If you'd rather we keep reporting in a single place, tell me which handler to keep and I'll update the PR."

- Privacy: "We scrub URL query params before sending them. The payload is intentionally minimal. We recommend server-side scrubbing and adding the telemetry policy to the docs."

- Production readiness: "This PR provides immediate crash protection and minimal reporting; for richer observability (source maps, grouping), I'll follow up with Sentry integration behind a config flag and instructions for source-map upload."

- Tests: "I can add Vitest unit tests for the ErrorBoundary in a follow-up PR; I kept this PR small so we can get the runtime protections in quickly."

Replace or extend these answers as needed for your PR conversation.
