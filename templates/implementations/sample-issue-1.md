# Issue #1: Add `/health` endpoint

**Status:** Complete
**Mode:** Hybrid
**Created:** 2026-04-27
**GitHub:** https://github.com/acme-org/acme-app/issues/1
**Repository:** acme-org/acme-app
**Traceability:** REQ-XX tags assigned (5 acceptance criteria)

> **Sample issue file** — synthetic example that walks through every section
> a real hybrid-mode issue file accumulates. Use this as a reference for what
> "good" looks like at each phase. Replace, don't extend, when you start
> a real issue.

---

## Original Requirements

Add a `/health` endpoint to the backend so the load balancer and
on-call monitoring can verify the service is up and ready to take
traffic. The endpoint must be cheap to call (no DB query, no
external network call) but informative enough to distinguish "alive
but degraded" from "fully ready."

**Requirements (REQ-XX tagged for traceability):**

- REQ-01: `GET /health` returns HTTP 200 with a JSON body when the
  service is up
- REQ-02: Response body includes `status` ("ok" | "degraded"),
  `version` (semantic version of the deployed build), and `uptime_s`
  (process uptime in seconds)
- REQ-03: Endpoint is unauthenticated (the load balancer can't carry
  credentials)
- REQ-04: Endpoint completes in < 50ms p95 (no DB or external calls
  on the hot path)
- REQ-05: A degraded state is reported (`status: "degraded"`) if any
  background worker has reported a heartbeat older than 60 seconds

---

## Codebase Exploration (Claude /scope)

**Agent:** Claude
**Date:** 2026-04-27

### Relevant Files

| File                                | Purpose                              | Lines |
| ----------------------------------- | ------------------------------------ | ----- |
| `backend/app/main.py`               | FastAPI app entry; route registry    | 1–80  |
| `backend/app/routers/__init__.py`   | Router registration                  | 1–25  |
| `backend/app/services/heartbeat.py` | Background worker heartbeat tracking | 1–60  |
| `backend/app/version.py`            | Build-time version constant          | 1–10  |
| `backend/tests/test_routers.py`     | Existing router test patterns        | 1–120 |

### Existing Patterns

- **Router registration:** Each router lives in `backend/app/routers/{name}.py`
  and is registered in `backend/app/routers/__init__.py` via
  `app.include_router(...)`. Follow this pattern.
- **Async route handlers:** All handlers are `async def`. No sync handlers.
- **Pydantic response models:** Every endpoint declares a Pydantic
  response model in `backend/app/models/`. Follow this pattern for the
  health response shape.
- **Test pattern:** `backend/tests/test_routers.py` uses `httpx.AsyncClient`
  with the FastAPI app fixture. New tests should follow.

### Key Code Snippets

```python
# backend/app/routers/__init__.py:5–18
from fastapi import FastAPI
from .users import router as users_router
from .sessions import router as sessions_router

def register_routers(app: FastAPI) -> None:
    app.include_router(users_router)
    app.include_router(sessions_router)
    # ← new health router goes here
```

```python
# backend/app/services/heartbeat.py:10–35
class HeartbeatTracker:
    def __init__(self) -> None:
        self._heartbeats: dict[str, datetime] = {}

    def record(self, worker_id: str) -> None:
        self._heartbeats[worker_id] = datetime.utcnow()

    def stale_workers(self, threshold_s: int = 60) -> list[str]:
        cutoff = datetime.utcnow() - timedelta(seconds=threshold_s)
        return [w for w, ts in self._heartbeats.items() if ts < cutoff]
```

### Applicable Constraints

- **02: API Endpoint Conventions** — All endpoints declared in
  `routers/`, registered in `routers/__init__.py`. Follow.
- **04: Auth Middleware Bypass List** — Unauthenticated endpoints must
  be added to the explicit bypass list in `backend/app/middleware/auth.py:42`
  (rather than checking auth and returning early in the handler).

### Database Schema (if relevant)

N/A — this endpoint does not touch the database.

### Security Considerations

- **Auth:** Endpoint is intentionally unauthenticated (REQ-03). Add
  to `backend/app/middleware/auth.py` bypass list explicitly so future
  refactors don't accidentally start requiring auth here.
- **Information disclosure:** Don't leak internal hostnames, env
  contents, or worker IDs in the response. The `degraded` flag is the
  only signal exposed externally; the _which_ worker is stale stays
  internal (logs only).
- **Rate limiting:** This endpoint is called by the load balancer
  every few seconds. Skip per-IP rate limiting on `/health`
  specifically — but keep the global rate limiter in place to catch
  abuse.

### Questions for Architect

- Should `/health` short-circuit return on the very first request
  before background workers have heartbeated, or wait until at least
  one heartbeat has been recorded? Trade-off: false-degraded at boot
  vs. delayed readiness signal.
- Should the response include a list of _which_ workers are stale, or
  just a boolean `degraded` flag? (Security-leaning preference is
  boolean only; observability-leaning preference is per-worker. Decide.)

---

_Next step: Open Codex and run `/architect 1` to create the architecture plan._

---

## Architecture Plan - Attempt 1

**Agent:** GPT
**Date:** 2026-04-27

### Approach

Add a single new router `backend/app/routers/health.py` exposing
`GET /health`. The handler reads the singleton `HeartbeatTracker`
instance, derives the `status` field from `stale_workers()`, and
returns a Pydantic-validated response. Process start time captured
at module import; uptime is `now - start_time`. Version is read
from the existing `backend/app/version.py` constant.

For the architect's open questions: at boot, treat zero recorded
heartbeats as `status: "ok"` (workers haven't booted yet, but the
API is up — that's the load-balancer-relevant signal). The
per-worker stale list stays in logs only; the public response is a
boolean `degraded` flag (no worker-ID disclosure).

### Implementation Steps

1. **`backend/app/routers/health.py`** - Create (implements REQ-01, REQ-02, REQ-03, REQ-05)
   - Define `HealthResponse` Pydantic model: `status: Literal["ok", "degraded"]`,
     `version: str`, `uptime_s: int`
   - Capture `START_TIME = time.monotonic()` at module level
   - Define `async def health(tracker: HeartbeatTracker = Depends(get_tracker)) -> HealthResponse`
   - Compute `stale = tracker.stale_workers(threshold_s=60)`; log stale
     worker IDs at INFO level if any; return status accordingly

2. **`backend/app/routers/__init__.py`** - Modify (implements REQ-01)
   - Import the new `health_router`
   - Register via `app.include_router(health_router)`

3. **`backend/app/middleware/auth.py`** - Modify (implements REQ-03)
   - Add `"/health"` to the explicit bypass list at line 42

4. **`backend/app/dependencies.py`** - Modify (implements REQ-04)
   - Add `get_tracker()` dependency that returns the singleton
     `HeartbeatTracker` (already instantiated at app startup)

5. **`backend/tests/test_health.py`** - Create
   - Test scenarios listed below

### Data Model Changes

None.

### API Changes

| Endpoint  | Method | Request | Response                                                      | Notes           |
| --------- | ------ | ------- | ------------------------------------------------------------- | --------------- |
| `/health` | GET    | (none)  | `{ status: "ok" \| "degraded", version: str, uptime_s: int }` | Unauthenticated |

### Frontend Changes

None.

### Test Strategy

1. `test_health_returns_200_ok_when_no_stale_workers` - verifies REQ-01, REQ-02
2. `test_health_returns_degraded_when_worker_heartbeat_older_than_60s` - verifies REQ-05
3. `test_health_unauthenticated_does_not_require_auth_header` - verifies REQ-03
4. `test_health_response_does_not_leak_worker_ids` - security check (boolean
   flag only; per-worker stale list must not appear in response body)
5. `test_health_p95_latency_under_50ms` - verifies REQ-04 (load 200
   requests in test, assert p95 < 50ms; informal because CI variance, but
   catches regressions)

### Risk Assessment

- **Risk: HeartbeatTracker singleton race conditions** — Multiple
  workers writing to `_heartbeats` concurrently could cause torn
  reads on the `stale_workers()` call. Mitigation: `_heartbeats` is
  a `dict` mutated only via `record()`, and `stale_workers()` reads
  via list comprehension over `.items()`. Python's GIL makes this
  safe for the dict-of-datetime case; no lock needed. Document the
  invariant in a code comment.
- **Risk: clock skew on degraded calculation** — If the clock jumps
  forward or backward, `stale_workers()` may flap. Acceptable risk
  for v1; revisit if false-positive degraded reports become noisy.

### Constraint Compliance

- Constraint 02 (API Endpoint Conventions): satisfied — new router in
  `routers/`, registered in `routers/__init__.py`
- Constraint 04 (Auth Middleware Bypass List): satisfied — `/health`
  added to the explicit bypass list at line 42

### Open Questions

None.

### Recommendation

**READY TO BUILD**

Architecture is straightforward; only addition over the requirements
is the no-auth-leakage discipline (boolean degraded flag, no per-
worker IDs in response body). Per-worker observability stays in logs
where ops can see it without exposing it externally.

---

## Implementation Notes - Attempt 1 (Claude /build)

**Agent:** Claude
**Date:** 2026-04-27
**Status:** SUCCESS

### Files Changed

| File                              | Action | Description                                    |
| --------------------------------- | ------ | ---------------------------------------------- |
| `backend/app/routers/health.py`   | Create | New `/health` router; handler + response model |
| `backend/app/routers/__init__.py` | Modify | Registered `health_router`                     |
| `backend/app/middleware/auth.py`  | Modify | Added `"/health"` to bypass list at line 42    |
| `backend/app/dependencies.py`     | Modify | Added `get_tracker()` dependency               |
| `backend/tests/test_health.py`    | Create | 5 tests covering REQ-01..REQ-05 + leak check   |

### Tests Written

| Test File                      | Count | Coverage Area                                                 |
| ------------------------------ | ----- | ------------------------------------------------------------- |
| `backend/tests/test_health.py` | 5     | REQ-01, REQ-02, REQ-03, REQ-04, REQ-05 + worker-ID leak check |

### Key Decisions

1. **Used `time.monotonic()` for uptime instead of `time.time()`** —
   Rationale: monotonic is immune to wall-clock adjustments (NTP, manual
   `date` set), so reported uptime can't go negative or jump.

2. **Logged stale worker IDs at INFO not WARNING** — Rationale: at INFO,
   ops can grep for them on demand, but they don't trip alert noise on
   every heartbeat blip. WARNING reserved for actually-actionable issues.

### For Reviewer

- The p95 latency test (REQ-04) is informal — 200 requests in pytest is
  not a real load test. Confirm this is acceptable for CI; otherwise
  recommend moving the latency assertion to a separate perf-test suite.
- Singleton-thread-safety reasoning relies on GIL semantics and the
  specific shape of `HeartbeatTracker._heartbeats`. If we ever switch
  to a process-pool worker (multiprocessing), this assumption breaks.
  Documented in a code comment but worth flagging here.

### Security-Sensitive Areas (Auto-Flagged)

- [ ] New/modified RLS policies: none
- [x] New API endpoints: `GET /health` — reviewer should verify the
      auth-middleware bypass is the _only_ unauthenticated path added
      and that no auth-bypass token leak exists
- [ ] New environment variables: none
- [ ] AI/LLM integration: none
- [ ] Payment/billing changes: none
- [ ] Database schema changes: none

---

_Next step: Open Codex and run `/review 1` for critic & security review._

---

## GPT Review - Attempt 1

**Agent:** GPT
**Date:** 2026-04-27

### Overall Assessment

Implementation matches the architecture plan closely. Code is small and
readable. Tests cover the acceptance criteria, including the leak-
prevention check. Two minor issues and one important issue worth
addressing before commit.

### Critical Issues (🔴)

None.

### Important Issues (🟡)

- **[backend/app/routers/health.py:14]** `START_TIME` captured at
  module import means uptime resets on hot-reload during dev but
  also when the app is imported by tests, which makes test
  `test_health_p95_latency_under_50ms` produce slightly inflated
  uptime readings on second-and-later runs of the same pytest
  session.
  - **Impact:** Test flake risk (low, but real). Production behavior
    is fine because the module is imported exactly once.
  - **Fix:** Move `START_TIME` capture into an `app.on_event("startup")`
    hook or attach it to `app.state.start_time`. Then `uptime_s` is
    `now - app.state.start_time`. Test fixtures can stub this.

### Minor Issues (🟢)

- **[backend/app/routers/health.py:28]** Logging stale worker IDs as
  a comma-joined string is fine for ops, but consider structured
  logging (`logger.info("stale_workers", extra={"worker_ids": stale})`)
  to play nicely with downstream log aggregators.
- **[backend/tests/test_health.py:67]** The p95 assertion uses
  `< 50ms` literal. Pull into a constant at the top of the file
  with a comment linking to REQ-04 — easier to audit.

### Security Analysis

#### Threat Areas Affected

API surface (new public endpoint, unauthenticated). Information
disclosure surface (response body shape).

#### OWASP Quick Check

- [x] Injection — N/A (no user input on this endpoint)
- [x] Broken Authentication — Endpoint intentionally unauthenticated
      per REQ-03; bypass list updated correctly. Verified that no
      _other_ paths were silently added to the bypass list.
- [x] Sensitive Data Exposure — Response is sanitized (boolean
      degraded flag, no worker IDs). Verified by
      `test_health_response_does_not_leak_worker_ids`.
- [x] Broken Access Control — Endpoint is intentionally public; no
      access control needed beyond the bypass entry.
- [x] Security Misconfiguration — Bypass entry is exact-match
      `"/health"`, not a prefix; no risk of `"/health/anything"`
      accidentally bypassing auth.

#### STRIDE Quick Assessment

| Category               | Applicable? | Finding                                                                                          |
| ---------------------- | ----------- | ------------------------------------------------------------------------------------------------ |
| Spoofing               | N/A         |                                                                                                  |
| Tampering              | N/A         |                                                                                                  |
| Repudiation            | N/A         |                                                                                                  |
| Information Disclosure | Yes         | Mitigated — response shape excludes worker IDs and any internal state.                           |
| Denial of Service      | Yes         | Acceptable — endpoint is cheap (no DB, no network) and global rate limiter still in front of it. |
| Elevation of Privilege | N/A         |                                                                                                  |

#### {Project}-Specific Checks

- [x] No new tables / RLS policies
- [x] No new admin/service-tier credential usage
- [x] Auth bypass list is explicit and exact-match

### Additional Tests Recommended

1. `test_health_bypass_list_is_exact_not_prefix` - verifies that
   `/healthcheck` or `/health/extra` requests still hit auth (not
   bypassed). Defense against future bypass-list refactor that
   accidentally introduces prefix matching.

### Frontend Flow Tests Recommended

None — this is a backend-only change.

### Positive Highlights

- The leak-prevention test (`test_health_response_does_not_leak_worker_ids`)
  is exactly the right shape for this kind of endpoint. Good defensive
  posture.
- Choice of `time.monotonic()` for uptime is correct and well-justified.

### AI Bias Observations

- **Acceleration:** None observed. Implementation matches the plan
  step-by-step.
- **Scope:** Slight scope creep — INFO-level structured logging
  decision was made during implementation rather than in the plan.
  Acceptable; documented in Key Decisions.
- **Over-optimization:** None observed.

### Requirement Traceability Verification

| REQ Tag | Requirement                                               | Implemented? | Tested?                                        |
| ------- | --------------------------------------------------------- | ------------ | ---------------------------------------------- |
| REQ-01  | GET /health returns 200 with JSON body when service is up | Yes          | Yes                                            |
| REQ-02  | Response includes status, version, uptime_s               | Yes          | Yes                                            |
| REQ-03  | Endpoint is unauthenticated                               | Yes          | Yes                                            |
| REQ-04  | < 50ms p95                                                | Yes          | Partial (informal test, not a real perf suite) |
| REQ-05  | Reports `degraded` if any worker heartbeat older than 60s | Yes          | Yes                                            |

### Recommendation

**REQUEST CHANGES**

One important issue (`START_TIME` test-flake risk) plus the additional
test recommendation. Both are small and quick to address. After fixing,
should be APPROVE on attempt 2.

---

## Implementation Notes - Attempt 2 (Claude /build)

**Agent:** Claude
**Date:** 2026-04-27
**Status:** SUCCESS

### Files Changed

| File                            | Action | Description                                                                                                             |
| ------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------- |
| `backend/app/routers/health.py` | Modify | Moved `START_TIME` capture into startup event, attached to `app.state`                                                  |
| `backend/app/main.py`           | Modify | Added `@app.on_event("startup")` hook to set `app.state.start_time`                                                     |
| `backend/tests/test_health.py`  | Modify | Added `test_health_bypass_list_is_exact_not_prefix`; pulled p95 literal into module-level constant `P95_BUDGET_MS = 50` |

### Tests Written

| Test File                      | Count Added | Coverage Area                                                             |
| ------------------------------ | ----------- | ------------------------------------------------------------------------- |
| `backend/tests/test_health.py` | 1           | New: `test_health_bypass_list_is_exact_not_prefix` (defends bypass scope) |

### Key Decisions

1. **Attached start_time to `app.state` rather than `app.on_event` closure** —
   Rationale: `app.state` is the FastAPI-blessed pattern for app-scoped
   mutable state and is testable via fixtures.

### For Reviewer

- All Important issues from Review Attempt 1 addressed.
- Both Minor issues from Attempt 1 addressed (structured logging +
  pulled-up constant).
- The new bypass-prefix test does what the reviewer asked for.

### Security-Sensitive Areas (Auto-Flagged)

- [x] New API endpoints: `/health` — verified bypass is exact-match
      via the new test
- (others as before — unchanged)

---

## GPT Review - Attempt 2

**Agent:** GPT
**Date:** 2026-04-27

### Previous Issues Verification

| Issue                                                  | Previous Severity | Status |
| ------------------------------------------------------ | ----------------- | ------ |
| START_TIME captured at module import → test flake risk | 🟡 Important      | FIXED  |
| Structured logging for stale worker IDs                | 🟢 Minor          | FIXED  |
| P95 literal in assertion                               | 🟢 Minor          | FIXED  |
| Missing test: bypass-list-is-exact-not-prefix          | (recommended)     | ADDED  |

### Overall Assessment

All Attempt 1 findings addressed. New test confirms bypass-list scope.
Implementation is ready to commit.

### Critical Issues (🔴)

None.

### Important Issues (🟡)

None.

### Minor Issues (🟢)

None.

### Security Analysis

No changes from Attempt 1; all checks still pass. The new
bypass-list-exact-match test strengthens the defensive posture.

### Additional Tests Recommended

None.

### Frontend Flow Tests Recommended

None.

### Positive Highlights

Clean address of Attempt 1 feedback. The `app.state.start_time`
choice is the correct FastAPI idiom and removes the test-flake risk
without over-engineering.

### AI Bias Observations

None.

### Requirement Traceability Verification

| REQ Tag | Requirement                                               | Implemented? | Tested?                                   |
| ------- | --------------------------------------------------------- | ------------ | ----------------------------------------- |
| REQ-01  | GET /health returns 200 with JSON body when service is up | Yes          | Yes                                       |
| REQ-02  | Response includes status, version, uptime_s               | Yes          | Yes                                       |
| REQ-03  | Endpoint is unauthenticated                               | Yes          | Yes                                       |
| REQ-04  | < 50ms p95                                                | Yes          | Partial (informal — acceptable per scope) |
| REQ-05  | Reports degraded if any worker heartbeat older than 60s   | Yes          | Yes                                       |

### Recommendation

**APPROVE**

Hand back to Claude for `/verify 1`.

---

## Verification & Commit (Claude /verify)

**Agent:** Claude
**Date:** 2026-04-27

### Issues Addressed

| Issue                                                  | Severity     | Fix Applied                                            |
| ------------------------------------------------------ | ------------ | ------------------------------------------------------ |
| START_TIME captured at module import → test flake risk | 🟡 Important | Moved to `app.state.start_time` via startup event hook |
| Structured logging for stale worker IDs                | 🟢 Minor     | Switched to `logger.info(..., extra={...})`            |
| P95 literal in assertion                               | 🟢 Minor     | Pulled into `P95_BUDGET_MS = 50` constant              |

### Issues Deferred

None.

### Test Results

- **Backend tests:** ✅ 6/6 passing (5 original + 1 new bypass-prefix test)
- **Frontend tests:** N/A (backend-only change)
- **Type check:** ✅ Pass (`mypy backend/app`)
- **Linting:** ✅ Pass (`ruff check backend/`)
- **Frontend (browser automation):** N/A
- **Migration Application Check:** N/A (no migration files in this issue)

### Acceptance Criteria

| Criterion | Met?           | Evidence                                                                      |
| --------- | -------------- | ----------------------------------------------------------------------------- |
| REQ-01    | Yes            | `test_health_returns_200_ok_when_no_stale_workers` passing                    |
| REQ-02    | Yes            | Pydantic `HealthResponse` model + same test verifies fields                   |
| REQ-03    | Yes            | `test_health_unauthenticated_does_not_require_auth_header` passing            |
| REQ-04    | Yes (informal) | `test_health_p95_latency_under_50ms` passing; documented as informal CI check |
| REQ-05    | Yes            | `test_health_returns_degraded_when_worker_heartbeat_older_than_60s` passing   |

### Commit

- **Branch:** `feature/health-endpoint`
- **Message:** `[#1] Add /health endpoint`
- **SHA:** `abc1234`
