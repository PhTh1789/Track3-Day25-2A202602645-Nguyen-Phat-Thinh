# Lab 25B - Game Day Report

**Ho ten / nhom:** Nguyen Phat Thinh / 2A202602645  
**Commit:** `be4d558` (report and grading artifacts commit)  
**Diem tu dong (`make grade`):** 84 / 88

## 1. Ket qua Game Day

| Bang | Client | Availability | P95 (s) | Amplification | Retry ratio | Cost/turn | Recovery (s) | PASS |
|---|---|---:|---:|---:|---:|---:|---:|---|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | No |
| G1 | resilient | 0.9993 | 4.771 | 1.061 | 0.001 | 2.092 | 0.0 | Yes |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | No |
| G2 | resilient | 0.9992 | 6.637 | 1.027 | 0.004 | 2.093 | 0.0 | Yes |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | No |
| G3 | resilient | 1.0000 | 4.668 | 1.059 | 0.030 | 2.143 | 0.0 | Yes |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | No |
| G4 | resilient | 1.0000 | 5.626 | 1.014 | 0.002 | 2.072 | 0.0 | Yes |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | - | No |
| G5 | resilient | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | - | Yes |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | No |
| G6 | resilient | 0.9765 | 5.990 | 1.039 | 0.005 | 2.245 | 0.0 | Yes |

The resilient client passed all six public tapes. It had zero retry-after violations,
zero client-error retries, zero incompatible routes and zero unnecessary failovers.

## 2. Client configuration

| Parameter | Value | Reason |
|---|---:|---|
| attempt_timeout_s | 8.0 | Leaves time for failover inside the 30 second user deadline; G4 P95 is 5.626s |
| max_attempts_per_provider | 2 | Allows one same-provider retry without amplification; G2 was 1.027 |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full jitter spreads retries while keeping the deadline usable |
| retry_budget_ratio | 0.1 | Keeps retries bounded; G3 retry ratio was 0.030 |
| breaker.window_size / min_calls | 20 / 10 | Avoids opening on a tiny sample while reacting during an incident |
| failure_rate_threshold | 0.5 | Opens on sustained provider failure |
| slow_call_s | 5.0 | Detects G4 degradation before timeout accumulation |
| open_duration_s | 10.0 | Gives a degraded provider time to recover |
| half_open_max_probes | 2 | Limits thundering herd during recovery |

Decision table:

| Error | Breaker | Retry/failover |
|---|---|---|
| 400 client error | release | Give up, never retry or fail over |
| 429 quota | release | Long provider cooldown, fail over |
| 429 rate limit | release | Honor retry-after and fail over |
| 529/503/5xx/timeout | failure | Retry same provider if budget and deadline allow, then fail over |

## 3. Experiments

1. The retry budget is a final defense even when the breaker is misconfigured. In
   normal configuration G2 amplification was 1.027 and G3 retry ratio was 0.030.
   With a breaker that never opens, disabling the budget would allow every transient
   error to create another attempt, while the budget caps retry traffic near 10%.
   The breaker reacts to provider health; the budget protects the whole system from
   retry amplification, so both are needed.

2. A 30 second attempt timeout is unsafe during G6 because an in-flight request can
   consume the entire user deadline and prevent fallback. G4 changes little because
   it has successful slow responses rather than provider failures. Timeout should be
   based on normal-tail latency, the user deadline and the time required for a useful
   fallback, not only on the provider maximum.

3. Removing tool capability filtering would allow `gamma-mini` to receive a tool
   request. That provider cannot perform tool calling, so the result is an
   incompatible model output rather than a valid fallback. Product-wise, this
   means the system spent money and latency on an answer that could never satisfy
   the requested operation.

## 4. Refund workflow

| Executor | Completion | Crashes | Duplicate refunds | Duplicate emails | Lost refunds | Overpaid | Wasted LLM/crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.000 | 81 | 128 | 81 | 0 | 1496700 cents | 1.123 |
| yours | 1.000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

An idempotency key alone prevents the payment service from executing the same
effect twice, but it does not prevent recomputing already completed LLM steps.
A checkpoint alone can resume work, but it cannot deduplicate a side effect whose
response was lost. The correct order is execute the step, then checkpoint its
successful result. Writing the checkpoint before payment could make a worker skip
payment after a crash and lose the refund.

## 5. Alerting and error budget

The burn-rate alerter passed the public SLI data with three pages, one ticket and
zero false pages. It detected checkout_bug after 27 minutes, provider_outage after
4 minutes, and degraded_evening after 291 minutes. It ticketed slow_leak after
1337 minutes without paging. The 30-day report has 25,901,755 requests, 55,618
bad requests, 99.7853% availability, 42.95% budget consumed and 57.05% remaining,
so releases are not frozen.

The workflow protects correctness of side effects; alerting protects detection and
operational response. Both are needed: a correct workflow can still be unavailable,
and an alert cannot make a payment exactly-once.

## 6. Postmortem - G2 with naive client

**Summary:** At t=120, alpha lost 75% of its GPU capacity. The three nested retry
layers multiplied load while timed-out requests continued consuming provider slots.
Although alpha recovered at t=300, abandoned queued work and continuing retries kept
the system in a metastable overloaded state.

**Impact:** Naive availability was 0.4560, amplification was 9.904, and recovery was
155 seconds after provider restore. The resilient client reached 0.9992 availability,
1.027 amplification and immediate measured recovery.

| Time | Event | Evidence |
|---:|---|---|
| 120 | Alpha loses 75% capacity | Incident trigger |
| 120-150 | Queue and retries grow | Attempts exceed successful calls in the generated CSV |
| 180 | Client timeouts become visible | Provider reports client timeouts |
| 240 | Retry storm is sustained | Naive amplification is 9.904 |
| 300 | Alpha fully restores | Scenario restore point |
| 300+ | System remains degraded | Naive recovery is 155s, not zero |

The trigger was the GPU capacity loss. The root cause was the three-layer retry
design without jitter, budget or breaker. Requests that had already timed out still
occupied provider slots, while new retries kept arriving faster than the recovered
capacity could drain the queue.

Helpful controls are bounded retries, breaker fail-fast behavior, capability-aware
fallback and virtual-time metrics. Making the timeout longer or adding more retries
would worsen the queue and hide the availability failure.

| # | Action | Type | Owner | Verification |
|---:|---|---|---|---|
| 1 | Keep one retry layer with a token budget | Prevent | Platform | G2 amplification <= 1.2 in CI |
| 2 | Alert on amplification and recovery time | Detect | SRE | Synthetic G2 page within the defined window |
| 3 | Use breaker plus half-open probe limits | Mitigate | Runtime | Hidden scenarios recover within 30s |

## 7. Remaining weakness

The client still pays more when beta carries spillover, especially in G5 where
cost per good turn reached 2.504. A production improvement would add adaptive
capacity-aware routing and an explicit cost budget while preserving capability and
deadline constraints.

## 8. Cache

| Client | Availability | FAQ availability | Hit ratio | Stale served | Wrong answers | Outdated | Cost/turn | PASS |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| resilient (no cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | No |
| cached | 0.9110 | 0.9628 | 0.578 | 150 | 0 | 4 | 0.965 | Yes |

| Cache parameter | Value | Reason |
|---|---:|---|
| similarity_threshold | 0.97 | Prevents the 181 wrong-question hits seen at 0.80 |
| ttl_s | 60 | Limits silent document-update exposure; outdated answers reduced to 4 |
| max_stale_s | 180 | Supplies 150 stale answers during outage without extending stale use indefinitely |

Personal questions are excluded because user-scoped caching still creates privacy,
authorization and invalidation risks; a personal answer must be recomputed against
current user data. During a cache-covered outage, alerting should use provider
health, GiveUp rate, stale-served rate and cache age, not only user availability.
