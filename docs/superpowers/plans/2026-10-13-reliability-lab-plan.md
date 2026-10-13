# Lab 25B Reliability System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement a production-grade resilient LLM client system, sliding-window circuit breaker, semantic cache with stale-if-error, exactly-once workflow executor, and SRE burn-rate alerting for ShopAgent to achieve 88/88 automated score and 12/12 report score on Game Day 25B.

**Architecture:** A layered fault-tolerance architecture where incoming requests pass through a semantic cache layer, then route via a capability-filtered fallback ladder through per-provider sliding-window circuit breakers governed by full jitter backoff and a global token-bucket retry budget. Workflows persist deterministic checkpoints to durable stores, and SRE alerts detect SLO burn rates across multiple rolling windows.

**Tech Stack:** Python 3.12, PyYAML, Pytest, Matplotlib, Discrete-event Virtual Simulator.

**Spec:** `docs/superpowers/specs/2026-10-13-reliability-lab-design.md`

## Global Constraints

- Never use real time (`time.time()`, `time.sleep()`); all timing uses discrete-event virtual time `now: float`.
- Do NOT edit forbidden directories or files: `src/gameday_lab/sim/`, `naive_client.py`, `workflows/services.py`, `workflows/runner.py`, `workflows/naive_executor.py`, `slo/data.py`, `slo/scoring.py`, `configs/scenarios*.yaml`, `tests/`.
- Do NOT hard-code incident times, tape names, document names, or inspect simulator private state.
- Strictly adhere to capability constraints: never send `task == "tool"` calls to models without tool support (e.g. `gamma-mini`).
- All tests must pass with `pytest` without warnings or failures.

## Review Focus

- HTTP 400 client errors must never be retried, must never trip breakers, and must never fail over.
- Rate limit 429 errors must never violate `retry-after` header delays.
- Half-open circuit breaker must strictly bound concurrent probe requests to `half_open_max_probes`.
- Exactly-once workflow must never record durable checkpoints before the action completes successfully.
- Semantic cache must never cache personal queries (`personal == True`) and must scope keys strictly by `prompt_version`.

---

### Task 1: Error Classification & Full Jitter Retry Budget (Part A)

**Files:**
- Modify: `src/gameday_lab/errors.py`
- Modify: `src/gameday_lab/retry.py`
- Test: `tests/test_errors_retry.py`

**Interfaces:**
- Consumes: `ProviderError`, `ErrorKind`
- Produces: `classify(err: ProviderError) -> ErrorKind`, `is_retryable(kind: ErrorKind) -> bool`, `counts_toward_breaker(kind: ErrorKind) -> bool`, `failover_helps(kind: ErrorKind) -> bool`, `FullJitterBackoff.delay(attempt: int, retry_after: float | None = None) -> float`, `RetryBudget.record_request(now: float) -> None`, `RetryBudget.try_acquire_retry(now: float) -> bool`.

- [ ] **Step 1: Inspect existing tests in `tests/test_errors_retry.py`**
Run: `.venv\Scripts\python -m pytest tests/test_errors_retry.py -q`
Expected: FAIL with `NotImplementedError`

- [ ] **Step 2: Implement error classification in `src/gameday_lab/errors.py`**
Implement:
- `classify(err)`: map 429 + `rate_limit_exceeded` to `RATE_LIMITED`; 429 + `insufficient_quota` to `QUOTA_EXHAUSTED`; 503/529 to `OVERLOADED`; 500 to `SERVER_ERROR`; 400 to `CLIENT_ERROR`; 0 or `client_timeout` to `TIMEOUT`.
- `is_retryable(kind)`: True for `RATE_LIMITED`, `OVERLOADED`, `SERVER_ERROR`, `TIMEOUT`. False for others.
- `counts_toward_breaker(kind)`: True for `OVERLOADED`, `SERVER_ERROR`, `TIMEOUT`. False for others.
- `failover_helps(kind)`: True for everything except `CLIENT_ERROR`.

- [ ] **Step 3: Implement `FullJitterBackoff` and `RetryBudget` in `src/gameday_lab/retry.py`**
Implement:
- `FullJitterBackoff.delay`: exponential backoff `uniform(0, min(cap, base * (2 ** (attempt - 1))))`. If `retry_after is not None`, return `max(computed, retry_after)`.
- `RetryBudget._refill(now)`: update tokens based on elapsed virtual time `dt * min_per_s`.
- `RetryBudget.record_request(now)`: refill, add `ratio` tokens up to `max_tokens`.
- `RetryBudget.try_acquire_retry(now)`: refill; if `tokens >= 1.0`, decrement 1 token and return True, else False.

- [ ] **Step 4: Verify Part A tests pass**
Run: `.venv\Scripts\python -m pytest tests/test_errors_retry.py -v`
Expected: 100% PASS

- [ ] **Step 5: Commit Part A changes**
```bash
git add src/gameday_lab/errors.py src/gameday_lab/retry.py
git commit -m "feat(reliability): implement error classification and retry budget"
```

---

### Task 2: Sliding-Window Circuit Breaker (Part B)

**Files:**
- Modify: `src/gameday_lab/breaker.py`
- Test: `tests/test_breaker.py`

**Interfaces:**
- Consumes: `Permit`, `BreakerState`, `Transition`, `SlidingWindowBreaker`
- Produces: `allow(now: float) -> Permit | None`, `on_success(permit: Permit, now: float, latency_s: float) -> None`, `on_failure(permit: Permit, now: float) -> None`, `release(permit: Permit, now: float) -> None`.

- [ ] **Step 1: Run breaker tests to verify initial failure**
Run: `.venv\Scripts\python -m pytest tests/test_breaker.py -q`
Expected: FAIL with `NotImplementedError`

- [ ] **Step 2: Implement `_record` in `src/gameday_lab/breaker.py`**
Implement sliding window tracking:
- Append `(failed, slow)` to `self._window`, popping oldest if length exceeds `window_size`.
- If `len(self._window) < self.min_calls`, return without tripping.
- Calculate failure rate `failures / total`. If `>= failure_rate_threshold` $\to$ `self._move(now, BreakerState.OPEN, "failure_rate")`.
- Else calculate slow rate `(slow and not failed) / total`. If `>= slow_rate_threshold` $\to$ `self._move(now, BreakerState.OPEN, "slow_call_rate")`.

- [ ] **Step 3: Implement `allow`, `on_success`, `on_failure`, `release` in `src/gameday_lab/breaker.py`**
Implement:
- `allow(now)`:
  - If OPEN: if `now - self.opened_at >= self.open_duration_s`, `_move(now, BreakerState.HALF_OPEN, "open_timeout_elapsed")` and fall through to HALF_OPEN logic; else return None.
  - If HALF_OPEN: if `self.probes_in_flight >= self.half_open_max_probes`, return None; else `probes_in_flight += 1` and return `Permit(self.name, self.generation, probe=True)`.
  - If CLOSED: return `Permit(self.name, self.generation, probe=False)`.
- `on_success(permit, now, latency_s)`:
  - If `permit.generation != self.generation`, ignore.
  - Compute `slow = latency_s >= self.slow_call_s`.
  - If HALF_OPEN: `probes_in_flight = max(0, probes_in_flight - 1)`. If `slow`, `_move(now, BreakerState.OPEN, "probe_slow")`. Else `probe_successes += 1`; if `probe_successes >= self.half_open_successes`, `_move(now, BreakerState.CLOSED, "probes_succeeded")`.
  - If CLOSED: `_record(now, failed=False, slow=slow)`.
- `on_failure(permit, now)`:
  - If `permit.generation != self.generation`, ignore.
  - If HALF_OPEN: `probes_in_flight = max(0, probes_in_flight - 1)`, `_move(now, BreakerState.OPEN, "probe_failed")`.
  - If CLOSED: `_record(now, failed=True, slow=False)`.
- `release(permit, now)`:
  - If `permit.generation != self.generation`, ignore.
  - If HALF_OPEN and `permit.probe`: `probes_in_flight = max(0, probes_in_flight - 1)`.

- [ ] **Step 4: Verify Part B tests pass**
Run: `.venv\Scripts\python -m pytest tests/test_breaker.py -v`
Expected: 100% PASS

- [ ] **Step 5: Commit Part B changes**
```bash
git add src/gameday_lab/breaker.py
git commit -m "feat(reliability): implement sliding-window circuit breaker"
```

---

### Task 3: ResilientClient & Fallback Ladder (Part C)

**Files:**
- Modify: `src/gameday_lab/client.py`
- Modify: `configs/gameday.yaml`
- Test: `tests/test_client.py`
- Benchmark: `python scripts/run_gameday.py --client resilient`

**Interfaces:**
- Consumes: `ProviderInfo`, `CallContext`, `Attempt`, `Send`, `GiveUp`, `SlidingWindowBreaker`, `RetryBudget`, `FullJitterBackoff`, `classify`
- Produces: `ResilientClient` class implementing `start`, `on_success`, `on_error`, `on_cancel`.

- [ ] **Step 1: Run client unit tests to verify initial failure**
Run: `.venv\Scripts\python -m pytest tests/test_client.py -q`
Expected: FAIL with `NotImplementedError`

- [ ] **Step 2: Implement `ladder`, `_route`, `start`, `on_success`, `on_error`, `on_cancel` in `src/gameday_lab/client.py`**
Implement:
- `ladder(task)`: filter `self.order` by capabilities (if `task == "tool"`, keep only providers with `supports_tools == True`).
- `_route(call, now, exclude_providers=())`: iterate candidates in `ladder(call.task)`; check cooldowns (rate limit and quota cooldowns); check breaker `allow(now)`. Return `(provider_name, permit)` or `(None, None)`.
- `start(call, now)`: register request in `self.budget.record_request(now)`. Initialize `call.state` (attempts per provider, tried providers). Route candidate. If no candidate available, return `GiveUp("no_healthy_provider")`. Else calculate attempt timeout and return `Send(provider, timeout_s, delay_s=0.0, tag=permit)`.
- `on_success(attempt, now, latency_s)`: report success to breaker with `attempt.tag`.
- `on_error(attempt, err, now)`:
  - Classify error using `classify(err)`.
  - If `CLIENT_ERROR`: release permit, return `GiveUp("client_error")`.
  - If `RATE_LIMITED`: release permit. Record cooldown until `now + (err.retry_after or 1.0)`. If call remaining time $< min\_time\_left\_s$, return `GiveUp("deadline_exceeded")`. Check if can retry on same provider vs failover.
  - If `QUOTA_EXHAUSTED`: release permit. Set provider cooldown for `quota_cooldown_s`. Fail over immediately.
  - If `OVERLOADED`, `SERVER_ERROR`, `TIMEOUT`: call `breaker.on_failure(attempt.tag, now)`. Check provider attempt count $< max\_attempts\_per\_provider$ and `budget.try_acquire_retry(now)`. If both True and breaker allows, retry with jitter backoff. Else fail over to next candidate in ladder.
- `on_cancel(call, tag, now)`: release permit if tag is Permit.

- [ ] **Step 3: Run client unit tests**
Run: `.venv\Scripts\python -m pytest tests/test_client.py -v`
Expected: PASS 6/6 tests

- [ ] **Step 4: Run Game Day simulations on 6 incident tapes and tune `configs/gameday.yaml`**
Run: `.venv\Scripts\python scripts/run_gameday.py --client resilient`
Tune parameters in `configs/gameday.yaml` if needed until all 6/6 tapes PASS:
- G1: availability $\ge 98\%$, 0 retry_after violations, amplification $\le 1.6$
- G2: availability $\ge 95\%$, amplification $\le 1.6$, recovery $\le 30$s, $\ge 80\%$ traffic returns to alpha
- G3: availability $\ge 98\%$, retry_ratio $\le 0.10$
- G4: availability $\ge 97\%$, p95 $\le 10$s
- G5: 0 retry on 400, $\le 10$ unneeded failovers, availability $\ge 78\%$
- G6: availability $\ge 95\%$, 0 tool calls to gamma-mini

- [ ] **Step 5: Run side-by-side comparison**
Run: `.venv\Scripts\python scripts/run_gameday.py --client both`
Expected: naive 0/6 PASS, resilient 6/6 PASS

- [ ] **Step 6: Commit Part C changes**
```bash
git add src/gameday_lab/client.py configs/gameday.yaml
git commit -m "feat(reliability): implement ResilientClient passing all 6 incident tapes"
```

---

### Task 4: Semantic Cache & Stale-if-error (Part G)

**Files:**
- Modify: `src/gameday_lab/cache.py`
- Modify: `configs/gameday.yaml` (cache section)
- Test: `tests/test_cache.py`
- Benchmark: `.venv\Scripts\python scripts/run_gameday.py --scenarios configs/scenarios_cache.yaml --client resilient,cached --out reports/cache`

**Interfaces:**
- Consumes: `Request`, `Answer`, `Serve`, `Send`, `GiveUp`, `Action`, `ResilientClient`
- Produces: `SemanticCache` (with `lookup`, `store`, `invalidate`), `CachingClient`.

- [ ] **Step 1: Run cache unit tests to verify failure**
Run: `.venv\Scripts\python -m pytest tests/test_cache.py -q`
Expected: FAIL with `NotImplementedError`

- [ ] **Step 2: Implement `SemanticCache` in `src/gameday_lab/cache.py`**
Implement:
- Disallow caching if `req.personal == True`.
- `lookup(req, now)`: Match only within identical `req.prompt_version`. Compute cosine similarity with stored embeddings. Find best candidate with similarity $\ge similarity\_threshold$. Check expiration: if $now \le created\_at + ttl\_s$, return `(answer, False)`. If $now \le created\_at + ttl\_s + max\_stale\_s$, return `(answer, True)`. Else expired, return None. Update LRU order on hit.
- `store(req, answer, now)`: If `req.personal`, do not store. Store entry with `(prompt_version, embedding, answer, created_at)`. Evict LRU entries if count exceeds `max_entries`.
- `invalidate(source)`: Remove all entries whose `answer.sources` contains `source`. Return count of removed entries.

- [ ] **Step 3: Implement `CachingClient` in `src/gameday_lab/cache.py`**
Implement:
- `start(call, now)`: Check `call.request`. If present: call `cache.lookup(call.request, now)`. If hit and fresh (`stale=False`), return `Serve(answer, stale=False)`. If miss or stale: save stale fallback in `call.state["stale_answer"]`, delegate to `inner.start(call, now)`.
- `on_success(attempt, now, latency_s)`: Delegate to `inner.on_success`. If `attempt.call.request` and `attempt.call.answer`: store in cache `cache.store(attempt.call.request, attempt.call.answer, now)`.
- `on_error(attempt, err, now)`: Delegate to `inner.on_error`. If inner returns `GiveUp`: check `attempt.call.state.get("stale_answer")`. If present, return `Serve(stale_answer, stale=True)`.
- `on_cancel(call, tag, now)`: Delegate to `inner.on_cancel`.
- `on_content_update(source, now)`: Invalidate cache entries with `cache.invalidate(source)`. Delegate to `inner.on_content_update`.

- [ ] **Step 4: Verify cache unit tests pass**
Run: `.venv\Scripts\python -m pytest tests/test_cache.py -v`
Expected: 100% PASS

- [ ] **Step 5: Run tape C1 and tune `configs/gameday.yaml` cache parameters**
Run: `.venv\Scripts\python scripts/run_gameday.py --scenarios configs/scenarios_cache.yaml --client resilient,cached --out reports/cache`
Tune `similarity_threshold` (e.g. 0.90 - 0.92 to avoid `other_question`), `ttl_s` (e.g. 60s to keep `outdated_answers <= 15`), `max_stale_s` (e.g. 180s to cover the 120s provider outage).
Verify C1 criteria:
- `wrong_answers == 0`
- `outdated_answers <= 15`
- `faq_availability >= 93%`
- `cache_hit_ratio >= 50%`

- [ ] **Step 6: Commit Part G changes**
```bash
git add src/gameday_lab/cache.py configs/gameday.yaml
git commit -m "feat(reliability): implement semantic cache with stale-if-error for C1 outage tape"
```

---

### Task 5: Exactly-Once Refund Workflow (Part D)

**Files:**
- Modify: `src/gameday_lab/workflows/executor.py`
- Test: `tests/test_executor.py`
- Benchmark: `.venv\Scripts\python scripts/run_workflows.py --executor both`

**Interfaces:**
- Consumes: `RefundRequest`, `WorkflowResult`, `DurableStore`, `STEPS`, `STEP_FNS`
- Produces: `idempotency_key(session_id: str, step: str) -> str`, `checkpoint_key(session_id: str, step: str) -> str`, `DurableExecutor.run(req: RefundRequest) -> WorkflowResult`.

- [ ] **Step 1: Run workflow unit tests to verify initial failure**
Run: `.venv\Scripts\python -m pytest tests/test_executor.py -q`
Expected: FAIL with `NotImplementedError`

- [ ] **Step 2: Implement deterministic keys and durable executor in `src/gameday_lab/workflows/executor.py`**
Implement:
- `idempotency_key(session_id, step)`: return deterministic string `f"{session_id}:{step}"`.
- `checkpoint_key(session_id, step)`: return `f"checkpoint:{session_id}:{step}"`.
- `_retry(fn, max_attempts=3)`: retry on `ServiceError` and `ServiceTimeout` with backoff; DO NOT catch `WorkerCrash`.
- `run(req)`:
  - For each step in `STEPS`:
    - Check if checkpoint exists in `self.store.get(checkpoint_key(req.session_id, step))`. If exists, load cached result.
    - If not: generate `idem_key = idempotency_key(req.session_id, step)`. Call `_retry(lambda: fn(ctx, idem_key))`.
    - AFTER step succeeds: save checkpoint to `self.store.set(checkpoint_key(req.session_id, step), result)`.
  - Return `WorkflowResult(success=True, ...)`.

- [ ] **Step 3: Run workflow unit tests**
Run: `.venv\Scripts\python -m pytest tests/test_executor.py -v`
Expected: 100% PASS

- [ ] **Step 4: Run 500 crash workflows simulation**
Run: `.venv\Scripts\python scripts/run_workflows.py --executor both`
Verify:
- 0 duplicate refunds
- 0 duplicate emails
- 0 lost refunds
- completion rate $\ge 99\%$
- wasted LLM calls per crash $\le 0.6$

- [ ] **Step 5: Commit Part D changes**
```bash
git add src/gameday_lab/workflows/executor.py
git commit -m "feat(reliability): implement durable exactly-once workflow executor"
```

---

### Task 6: SRE Burn-rate Alerting & Error Budget (Part E)

**Files:**
- Modify: `src/gameday_lab/slo/alerting.py`
- Test: `tests/test_alerting.py`
- Benchmark: `.venv\Scripts\python scripts/run_slo.py --alerter both`

**Interfaces:**
- Consumes: `SloRule`, `AlertEvent`, `ErrorBudgetReport`
- Produces: `WindowSums`, `burn_rate(bad: int, total: int, slo: float) -> float`, `BurnRateAlerter.evaluate(records: list) -> list[AlertEvent]`, `error_budget_report(...) -> ErrorBudgetReport`.

- [ ] **Step 1: Run alerting unit tests to verify initial failure**
Run: `.venv\Scripts\python -m pytest tests/test_alerting.py -q`
Expected: FAIL with `NotImplementedError`

- [ ] **Step 2: Implement `WindowSums`, `burn_rate`, `evaluate`, `error_budget_report` in `src/gameday_lab/slo/alerting.py`**
Implement:
- `WindowSums.__init__(values)`: build prefix sum array `prefix = [0] * (N + 1)`.
- `WindowSums.sum(end_inclusive, length)`: `start = max(0, end_inclusive - length + 1); return prefix[end_inclusive + 1] - prefix[start]`.
- `burn_rate(bad, total, slo)`: `(bad / total) / (1.0 - slo)` if `total > 0` else 0.0.
- `BurnRateAlerter.evaluate(records)`:
  - For each rule (multi-window: both long and short windows must exceed burn rate threshold):
  - Mark active minutes. Merge contiguous active minutes into `AlertEvent(rule_name, start_idx, end_idx)`.
- `error_budget_report(records, slo, period_days)`:
  - Calculate total bad, total calls. Allowed bad $= total \times (1 - slo)$.
  - Spent percentage $= bad / allowed \times 100$. Freeze release if spent $> 100\%$.

- [ ] **Step 3: Run alerting unit tests**
Run: `.venv\Scripts\python -m pytest tests/test_alerting.py -v`
Expected: 100% PASS

- [ ] **Step 4: Run SLO evaluation benchmark on 30 days SLI data**
Run: `.venv\Scripts\python scripts/run_slo.py --alerter both`
Verify:
- Pages on all major incidents within SLA.
- Tickets on slow leak.
- 0 false pages on blips.

- [ ] **Step 5: Commit Part E changes**
```bash
git add src/gameday_lab/slo/alerting.py
git commit -m "feat(reliability): implement multi-window burn rate alerting and error budget reporting"
```

---

### Task 7: Full Autograder Run, Experiments, and Postmortem Report (Part F)

**Files:**
- Run: `scripts/grade.py`
- Run: `scripts/plot_timeseries.py`
- Create: `reports/REPORT.md` (copied from `reports/REPORT_TEMPLATE.md` and filled with data)

- [ ] **Step 1: Run autograder to verify 88/88 score**
Run: `.venv\Scripts\python scripts/grade.py`
Verify: `reports/scoreboard.json` has 88/88 points.

- [ ] **Step 2: Generate G2 overload storm plot**
Run: `.venv\Scripts\python scripts/plot_timeseries.py --scenario G2_overload_storm`
Verify: `reports/gameday/G2_overload_storm.png` generated.

- [ ] **Step 3: Run the 3 experiments required by Section 3 of the report**
- Experiment 1: Disable retry budget vs enable retry budget under misconfigured breaker on G2, G3.
- Experiment 2: `attempt_timeout_s: 30` on G4, G6.
- Experiment 3: Remove tool filtering from `ladder()` on G6.
- Cache Experiments: similarity threshold variation (0.80 vs tuned vs 0.97), TTL variation (30s vs 60s vs 300s), stale-if-error disabled (`max_stale_s: 0`).

- [ ] **Step 4: Fill `reports/REPORT.md` completely**
Fill all 8 sections:
- Section 1: Full Game Day results table (naive vs resilient across G1-G6).
- Section 2: Client configuration parameters justified with empirical data + ASCII decision diagram.
- Section 3: 3 experiments with before/after numbers and mechanism explanations.
- Section 4: Workflow table and durable execution analysis.
- Section 5: Alerting table and error budget freeze decision.
- Section 6: Blameless postmortem for G2 with $\ge 5$ timestamped milestones from CSV, trigger vs root cause, metastable failure dynamics, and 3 actionable items (prevent, detect, mitigate).
- Section 7: Remaining weakness analysis.
- Section 8: Cache results table, parameter justification, 3 cache experiments, and answers to caching personal queries / alerting during outage.
Fill student name, student ID, commit hash, and score.

- [ ] **Step 5: Final review and commit**
```bash
git add reports/REPORT.md reports/scoreboard.json reports/gameday/*.png
git commit -m "docs(report): complete comprehensive lab report and G2 blameless postmortem"
```
