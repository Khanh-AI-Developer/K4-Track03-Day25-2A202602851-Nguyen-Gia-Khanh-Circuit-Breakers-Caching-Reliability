# Design Spec: Circuit Breakers, Caching & Reliability for Production Agents (Lab 25B)

**Date:** 2026-10-13  
**Status:** Approved  
**Topic:** Reliability Engineering for ShopAgent (Game Day 25B)

---

## 1. Context and Problem Statement

ShopAgent is a customer service agent handling refund workflows. Each user turn requires 2 LLM calls: `tool` (requiring function/tool calling capability) and `chat`. Users wait at most 30 seconds for the entire turn.

The v1 implementation ([NaiveClient](file:///f:/VinUniversity/Schedule_Study_VinUniversity/Period_Two/Day10/K4-Track03-Day25-2A202602851-Nguyen-Gia-Khanh-Circuit-Breakers-Caching-Reliability/src/gameday_lab/naive_client.py)) has 3 nested layers of retries (app × gateway × SDK = up to 36 attempts/call), fixed delays without jitter, no circuit breaker, and no fallback ladder. In a real incident, a 3-minute provider brown-out was transformed into a 5-minute total system outage due to self-amplified load (metastable failure).

We are tasked with replacing [NaiveClient](file:///f:/VinUniversity/Schedule_Study_VinUniversity/Period_Two/Day10/K4-Track03-Day25-2A202602851-Nguyen-Gia-Khanh-Circuit-Breakers-Caching-Reliability/src/gameday_lab/naive_client.py) with a fault-tolerant client system that reliably passes 6 incident tapes (G1–G6) and 1 cache outage tape (C1), implements exactly-once refund workflows, builds SRE multi-window burn-rate alerting, and produces a complete blameless postmortem.

---

## 2. System Architecture & Components

### 2.1 Part A: Error Classification & Retry Control
- **File:** `src/gameday_lab/errors.py`, `src/gameday_lab/retry.py`
- **Error Classifier (`classify`):**
  - Status 429 + code `rate_limit_exceeded` $\to$ `RATE_LIMITED` (retryable=True, counts_toward_breaker=False, failover_helps=True)
  - Status 429 + code `insufficient_quota` $\to$ `QUOTA_EXHAUSTED` (retryable=False, counts_toward_breaker=False, failover_helps=True)
  - Status 529 / 503 $\to$ `OVERLOADED` (retryable=True, counts_toward_breaker=True, failover_helps=True)
  - Status 500 $\to$ `SERVER_ERROR` (retryable=True, counts_toward_breaker=True, failover_helps=True)
  - Status 400 $\to$ `CLIENT_ERROR` (retryable=False, counts_toward_breaker=False, failover_helps=False)
  - Status 0 $\to$ `TIMEOUT` (retryable=True, counts_toward_breaker=True, failover_helps=True)
- **Full Jitter Backoff (`FullJitterBackoff`):**
  - Computes exponential backoff $delay = \text{uniform}(0, \min(cap, base \times 2^{attempt - 1}))$.
  - Enforces `retry_after`: $delay = \max(delay, retry\_after)$ if header present.
- **Retry Budget (`RetryBudget`):**
  - Token bucket replenishes by elapsed virtual time: $tokens += dt \times min\_per\_s$.
  - First-time requests deposit `ratio` tokens (capped at `max_tokens`).
  - Retries spend 1 token (only allowed if $tokens \ge 1.0$).

### 2.2 Part B: Sliding-Window Circuit Breaker
- **File:** `src/gameday_lab/breaker.py`
- **SlidingWindowBreaker:**
  - Maintains a sliding deque of $(failed, slow)$ of maximum length `window_size`.
  - When length $\ge min\_calls$:
    - If failure rate $\ge failure\_rate\_threshold$ $\to$ transitions to OPEN (`"failure_rate"`).
    - Elif slow rate ($slow \land \neg failed$) $\ge slow\_rate\_threshold$ $\to$ transitions to OPEN (`"slow_call_rate"`).
  - CLOSED: returns regular `Permit(name, generation, probe=False)`.
  - OPEN: if $now - opened\_at \ge open\_duration\_s$, transitions to HALF_OPEN (`"open_timeout_elapsed"`).
  - HALF_OPEN: admits at most `half_open_max_probes` concurrent probes in flight. Returns `Permit(name, generation, probe=True)`.
  - On probe success: decrements probes in flight; if slow $\to$ OPEN (`"probe_slow"`); else increments probe successes and closes (`"probes_succeeded"`) once threshold reached.
  - On probe failure: decrements probes in flight and transitions to OPEN (`"probe_failed"`).
  - Stale permits (permit generation != current generation) are safely ignored. Non-informative errors release probe slots without recording.

### 2.3 Part C: ResilientClient & Fallback Ladder
- **File:** `src/gameday_lab/client.py`
- **Capability Filtering (`ladder`):** Filters configured fallback order to exclude models lacking required capabilities (e.g., `gamma-mini` does not support tool calling, so it is omitted when `task == "tool"`).
- **Execution & Failover:**
  - `start()`: Records request into retry budget; attempts admission with primary provider breaker permit. Sends attempt with configured timeout.
  - `on_success()`: Records latency to provider breaker; finishes call.
  - `on_error()`:
    - 400 Client Error $\to$ GiveUp immediately (`"client_error"`).
    - Rate Limited $\to$ records provider rate limit cooldown; retries or fails over if time and budget permit.
    - Quota Exhausted $\to$ marks provider on long cooldown; immediately fails over to next candidate.
    - Overloaded / Server Error / Timeout $\to$ records failure to breaker; retries with jitter if within max attempts and budget; else fails over to next available provider.

### 2.4 Part G: Semantic Cache & Stale-if-error
- **File:** `src/gameday_lab/cache.py`
- **SemanticCache:**
  - Scoped strictly by `prompt_version`.
  - Ignores personal user queries (`request.personal == True`).
  - Cosine similarity matching: $\frac{A \cdot B}{\|A\|_2 \|B\|_2} \ge similarity\_threshold$.
  - TTL freshness vs stale window: Fresh within $t \le created\_at + ttl\_s$; stale if $t \le created\_at + ttl\_s + max\_stale\_s$.
  - Invalidation by source for document updates. LRU eviction bounded by `max_entries`.
- **CachingClient:**
  - Intercepts requests: fresh cache hit returns `Serve(answer, stale=False)`.
  - Cache miss delegates to `ResilientClient`.
  - On provider success: caches answer if request is non-personal.
  - On provider `GiveUp`: serves stale cached answer (`Serve(answer, stale=True)`) if available within `max_stale_s`.

### 2.5 Part D: Exactly-Once Refund Workflow
- **File:** `src/gameday_lab/workflows/executor.py`
- **Deterministic Keys:** `idempotency_key(session_id, step)` constructed deterministically (e.g. `f"{session_id}:{step}"`), never random UUIDs.
- **Execution Safety:**
  - Check `DurableStore` checkpoint first.
  - If uncompleted: invoke service with idempotency key, retrying transient service errors.
  - Record checkpoint in `DurableStore` ONLY AFTER step successfully completes.
  - Never catch `WorkerCrash`.

### 2.6 Part E: SRE Burn-rate Alerting & Error Budget
- **File:** `src/gameday_lab/slo/alerting.py`
- **$O(1)$ Window Aggregation (`WindowSums`):** Computes prefix sums for rapid multi-window slice queries.
- **Burn Rate:** $(\text{bad} / \text{total}) / (1 - \text{SLO})$.
- **Multi-window Rules:**
  - 14.4× (1h long, 5m short) $\to$ Page
  - 6× (6h long, 30m short) $\to$ Page
  - 1× (3d long, 6h short) $\to$ Ticket
- **Consolidation:** Merges consecutive burning minutes into single `AlertEvent`.
- **Error Budget Report:** Calculates total 30-day budget consumed and flags release freeze if $> 100\%$.

### 2.7 Part F: Report & Postmortem
- **File:** `reports/REPORT.md`
- Generates timeseries plots and gathers empirical metrics from `reports/gameday/`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/`.
- Authors a blameless postmortem for incident G2 with precise timestamps, trigger vs root cause analysis, metastable failure dynamics, and concrete action items.

---

## 3. Success Verification
- `pytest tests/test_errors_retry.py` (Part A passes 100%)
- `pytest tests/test_breaker.py` (Part B passes 100%)
- `pytest tests/test_client.py` (Part C passes 100%)
- `python scripts/run_gameday.py --client both` (All 6 incident tapes PASS)
- `pytest tests/test_cache.py` and `make cache` (C1 passes with 0 wrong answers)
- `pytest tests/test_executor.py` and `make workflows` (100% completion, 0 duplicates, 0 lost refunds)
- `pytest tests/test_alerting.py` and `make slo` (All major incidents alerted, 0 false pages)
- `python scripts/grade.py` achieves 88/88 automated score.
