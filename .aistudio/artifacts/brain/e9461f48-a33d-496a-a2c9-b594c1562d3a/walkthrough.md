# Security Summary

Following the PostgreSQL DBMS integration, a full security and reliability audit was completed across authentication, multi-tenant authorization, database access, simulated vs. BLE measurement provenance, clinical PDF generation, audit logging, and API rate limiting. Every fix preserves existing database records and active Cloud SQL + Firebase Auth functionality.

## Key Hardening Changes Implemented

1. **Production Session Secret Enforcement & Logout Revocation**:
   - Local bearer tokens (`hnloc.*`) now include a unique token identifier (`jti`) and are revoked server-side immediately when signing out via `POST /api/auth/logout`.
   - In production (`NODE_ENV=production`), the backend refuses to sign or verify local session tokens unless `AUTH_SESSION_SECRET` is configured with at least 32 characters, eliminating silent ephemeral secret fallbacks.
2. **Registration & Suspended Clinic Protection**:
   - Public self-registration (`POST /api/auth/register`) explicitly blocks `role: 'admin'`, no longer auto-promotes bootstrap admin emails via local password creation, and wraps user + clinic creation inside an atomic database transaction.
   - Users belonging to a suspended hospital or clinic are blocked (`HTTP 403`) on both login and authenticated API calls.
3. **Simulated vs. Physical BLE Provenance Integrity**:
   - Simulated identifiers (`SIM-*`) cannot be registered with `connectionType: 'ble'`.
   - Measurements marked `ble_device` require an authorized physical BLE device record, are blocked on `coming_soon` / `unavailable` channels, and cannot be linked to simulated devices or simulated test sessions.
   - Server-derived `isSimulated` flags ignore client overrides, and `POST /api/devices/:id/events` derives `connectionMode` strictly from the stored device record.
4. **Report Scope & Privacy Protection**:
   - `POST /api/reports` validates every requested `measurementId` against the caller's authorized scope (rejecting partial/mixed unauthorized lists with `HTTP 403`), enforces `sessionId` ownership (`HTTP 403`), and runs inside a transaction.
   - Clinic patient PDF exports no longer display the operator's personal date of birth or email in the patient details card.
5. **Rate Limiting, Transactions, & Reliability**:
   - Added sliding-window rate limiting (`HTTP 429` + `Retry-After`) on authentication and general `/api/*` routes, using `trust proxy = 1` so spoofed `X-Forwarded-For` headers cannot bypass limits.
   - Configured PostgreSQL pool `idleTimeoutMillis` (30s) and `statement_timeout` (15s), wrapped multi-step writes in transactions, handled concurrent `idempotencyKey` race collisions (`23505`) gracefully, and mapped malformed JSON to `HTTP 400` instead of `500`.
   - Restricted `POST /api/audit-logs` to an allowlist of client-side telemetry actions so users cannot forge privileged security log entries.

---

## PoC Verification

All checks below ran live against the restarted local server on port 3000 backed by Cloud SQL PostgreSQL (`cloud_sql_development_database`), followed by cleanup of all temporary synthetic test records so baseline row counts remained untouched.

| Verification Check | Outcome | Observed Result |
| --- | --- | --- |
| Health & Test Catalog (`GET /api/health`, `GET /api/test-types`) | Passed | `HTTP 200` (`database=connected`, 15 seeded parameters) |
| Production Session Secret Guard (`NODE_ENV=production`, unset secret) | Passed | Signing threw error; verification returned `null` (fail closed) |
| Admin Self-Registration Rejection (`POST /api/auth/register` with `role: 'admin'`) | Passed | Rejected with `HTTP 403` |
| Token Tampering & Logout Revocation | Passed | Altered signature returned `HTTP 401`; logged-out token immediately returned `HTTP 401` |
| Cross-Consumer & Cross-Hospital Isolation (`patients`, `devices`, `test-sessions`, `calibrations`, `reports`) | Passed | Unauthorized cross-user and cross-clinic probes returned `HTTP 403` / `HTTP 404` |
| Mixed-ID & Unauthorized `sessionId` Report Poisoning (`POST /api/reports`) | Passed | Mixing own + another clinic's `measurementId` or unauthorized `sessionId` returned `HTTP 403` |
| Simulated vs. BLE Provenance Enforcement | Passed | `SIM-*` as BLE, `ble_device` without BLE hardware, `ble_device` on simulated devices/`crp`, and `real` sessions on simulated devices all returned `HTTP 400`; `isSimulated: false` spoof was overridden to `true` |
| Client Audit Log Forgery Rejection (`POST /api/audit-logs`) | Passed | Attempting `PERMISSION_CHANGED` returned `HTTP 400`; allowed client event `SIMULATED_FAILURE_HANDLED` returned `HTTP 201` |
| Concurrent Idempotency Race (`Promise.all` x5 identical `idempotencyKey`) | Passed | Strictly 1 `HTTP 201` created and 4 `HTTP 200` (`deduplicated: true`) returned |
| Auth Brute-Force Rate Limiting (`POST /api/auth/login` burst) | Passed | Returned `HTTP 429` with `Retry-After` header |
| Malformed JSON Handling (`Content-Type: application/json`, `{invalid_json`) | Passed | Returned `HTTP 400` (`Malformed JSON request payload.`) instead of `500` |
| TypeScript & Production Build (`tsc --noEmit`, `vite build`) | Passed | Both exited cleanly with zero errors |
