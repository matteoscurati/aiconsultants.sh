> Report from the real run shown in the video on aiconsultants.sh (ai-consultants 5.3.0, 2026-10-21). The Coverage Union section is re-rendered from the same `synthesis.json` with the 5.3.1 report fix; 5.3.0 printed "coverage data unavailable" there because of a jq quoting bug. Nothing else is changed.

# AI Consultation Report

**Date**: 2026-10-21 08:03:53
**Directory**: /Users/matteoscurati/webhook-service
**Version**: AI Consultants v5.3.0
**Category**: API_DESIGN
**Outcome**: MET (3/3 responded, quorum=2)

---

## Question

What can go wrong when a service retries failed webhook deliveries?

## Quick Summary

| Consultant | Quality | Confidence | Approach |
|------------|---------|------------|----------|
| Gemini | structured | 10/10 | Bulkheaded Asynchronous Webhoo |
| Codex | structured | 9/10 | Bounded at-least-once delivery |
| Mistral | structured | 8/10 | At-Least-Once Webhook Delivery |

---

## Automatic Synthesis

**Synthesis Provider**: claude

**Coverage Integrity**: MET (37/37 normalized source IDs represented)
**Audited Fields**: summary, pros, cons, alternatives, caveats, references (MET applies only to these fields)
**Coverage Input Truncated**: false
**Truncated Consultants**: none

> Every normalized source ID from the audited fields is represented exactly once.

**Recommended Approach**: Bounded, durable, isolated at-least-once retry with receiver-side idempotency

Treat webhook retry as an at-least-once distributed delivery problem: persist every attempt durably, retry with capped exponential backoff and full jitter, honor Retry-After, and isolate endpoints and tenants with bulkheads and circuit breakers. Accept that duplicates and reordering are unavoidable and push deduplication and sequencing to receivers via idempotency keys, monotonic event IDs, and versioned payloads, while quarantining poison messages and securing every re-sent attempt.

### Coverage Union

- **evidence**: Overall framing: retrying failed webhooks converts transient failures into duplicate actions, out-of-order state updates, retry storms that amplify outages, poison-message queue blockage, and multi-tenant resource starvation; mitigated by bounded jittered backoff, durable delivery records, idempotency, bulkheads, and circuit breakers. — raised by Codex, Gemini, Mistral
- **recommendation**: Retries recover automatically from transient network and endpoint failures and raise overall delivery success rates. — raised by Codex, Mistral
- **recommendation**: Build the retry mechanism on established queue and database patterns rather than bespoke loops. — raised by Codex
- **recommendation**: Make failed deliveries observable and replayable, including alerting on dead letter queue depth. — raised by Codex, Mistral
- **risk**: Duplicate deliveries remain possible under at-least-once retry; deduplication and ordering responsibility is shifted onto the receiving consumer, which must be idempotent and tolerate concurrent delivery. — raised by Codex, Gemini, Mistral
- **recommendation**: Implement idempotency keys with appropriate TTLs so receivers can safely discard duplicates without unbounded dedup storage. — raised by Mistral
- **edge_case**: Exactly-once business effects cannot be guaranteed by retries alone; they require coordination with every affected downstream system. — raised by Codex
- **trade_off**: Robust retry requires durable state, retention policies, monitoring, partitioned message queues, distributed circuit-breaker state, and dead letter queue management, raising infrastructure complexity. — raised by Codex, Gemini
- **trade_off**: Backoff and tripped circuit breakers increase delivery latency and introduce eventual-consistency delays for receivers. — raised by Codex, Gemini
- **edge_case**: Which status codes are retryable depends on the endpoint contract; not every 4xx is permanent and not every 5xx is transient, so naive status-class rules misclassify failures. — raised by Codex
- **edge_case**: An HTTP success response only confirms what the receiver contract promises, which may be acceptance rather than completed processing, so a 2xx can mask downstream failure. — raised by Codex
- **recommendation**: Isolate tenant failure domains with bulkheaded queues and per-tenant concurrency limits to prevent one failing endpoint from starving others (noisy neighbor). — raised by Gemini
- **risk**: Thundering herd / retry storm: synchronized or predictable retry schedules without jitter overwhelm a recovering endpoint and can exhaust the sender's worker pools and connection limits. — raised by Mistral
- **recommendation**: Use exponential backoff with full jitter plus per-endpoint circuit breakers to protect downstream integration points from retry storms. — raised by Gemini, Mistral
- **risk**: Replay attack surface: retrying with retained, long-lived signatures lets a captured delivery be replayed against the receiver. — raised by Mistral
- **recommendation**: Secure replays with immutable event IDs, payload versioning, per-attempt HMAC signing, and short-lived signatures/timestamps. — raised by Gemini, Mistral
- **risk**: Out-of-order delivery: a retried older event can arrive after a newer one and regress receiver state or disrupt sequencing. — raised by Mistral
- **edge_case**: Strict FIFO ordering and high-throughput concurrent retries are mutually exclusive; enforcing per-endpoint ordering creates head-of-line blocking behind a failing event. — raised by Gemini
- **edge_case**: Outbound dispatchers remain exposed to SSRF and DNS rebinding on each retry (the target can re-resolve to a private IP) unless egress goes through a hardened proxy with private-range blocklists. — raised by Gemini
- **edge_case**: Honoring HTTP 429 Retry-After headers requires dynamic delayed scheduling; static fixed-delay queues cannot respect the receiver's requested wait. — raised by Gemini
- **risk**: Poison messages that fail deterministically block queue processing for everything behind them. — raised by Mistral
- **recommendation**: Quarantine poison messages immediately into a dead letter path instead of continuing to retry them in the main queue. — raised by Mistral
- **risk**: Resource exhaustion from unbounded retry queues (memory, storage, connections) during prolonged downstream outages. — raised by Mistral
- **recommendation**: Cap maximum retry attempts and maximum delay so retries are bounded in time and volume. — raised by Mistral
- **risk**: Compliance and privacy risk from retaining payloads in dead letter queues beyond permitted retention or jurisdiction limits. — raised by Mistral
- **recommendation**: Validate payload freshness before retrying; a stale event may be superseded or no longer valid and should be dropped or regenerated rather than re-sent. — raised by Mistral

### Risk Assessment

**Overall Risk**: high

- **Duplicate deliveries cause repeated business side effects at the receiver.** (high): Immutable event IDs and idempotency keys with TTLs; document at-least-once semantics to receivers.
- **Retried older events arrive after newer ones and regress receiver state.** (high): Monotonic sequence numbers or versioned payloads so receivers discard stale updates; avoid strict FIFO that causes head-of-line blocking.
- **Retry storms / thundering herd amplify an outage at a recovering endpoint and exhaust sender resources.** (high): Exponential backoff with full jitter, per-endpoint circuit breakers, honor Retry-After with dynamic scheduling.
- **Poison messages or one failing tenant block or starve delivery for others.** (medium): Immediate quarantine to DLQ after deterministic failure; tenant bulkheads and per-tenant concurrency limits.
- **Unbounded retry queues exhaust memory, storage, or connections during prolonged outages.** (medium): Cap attempts and max delay; bounded queues with DLQ overflow and depth alerts.
- **Replay attacks using retained long-lived signatures on re-sent payloads.** (medium): Per-attempt short-lived HMAC signatures with timestamps and event IDs receivers can dedupe.
- **SSRF and DNS rebinding when a retry re-resolves the target hostname to a private address.** (medium): Route egress through a hardened proxy with private-IP blocklists and validate on every attempt.
- **Misclassifying failures (treating permanent 4xx as transient or vice versa) and trusting 2xx as processed.** (medium): Define per-endpoint retry contract for status codes and success semantics; expose replay for ambiguous outcomes.
- **Compliance exposure from payloads retained in dead letter queues.** (low): Retention policies, encryption, and purge schedules for DLQ data.
- **Stale payloads re-sent after being superseded.** (low): Validate payload freshness before each retry and drop or regenerate stale events.

---

## Individual Responses

### Gemini

**Persona**: The Architect | **Confidence**: 10/10 | **Approach**: Bulkheaded Asynchronous Webhook Dispatch with Circuit Breaking & Idempotency

Retrying failed webhooks introduces distributed systems hazards including duplicate processing, out-of-order state regressions, retry storms, and multi-tenant resource starvation. Naive retry loops can overwhelm recovering downstream consumers and exhaust the sender's worker pools and connection limits. Enterprise architectures mitigate these failures via idempotency keys, exponential backoff with jitter, tenant bulkheads, and circuit breakers.


### Codex

**Persona**: The Pragmatist | **Confidence**: 9/10 | **Approach**: Bounded at-least-once delivery with idempotent processing

Retries can turn a temporary webhook failure into duplicate actions, out-of-order updates, or an outage amplified by retry traffic. Use bounded retries with backoff and jitter, durable delivery records, and receiver-side idempotency.


### Mistral

**Persona**: The Devil's Advocate | **Confidence**: 8/10 | **Approach**: At-Least-Once Webhook Delivery with Retries

Retrying failed webhooks creates duplicate delivery risks that violate idempotency expectations, enables thundering herd attacks via predictable patterns, and can trap poison messages that block entire delivery queues. Without jittered backoff and proper isolation, retry storms cascade failures across dependent systems.


---

## Output Files

- Context: `/Users/matteoscurati/.cache/ai-consultants/consultations/20261021_080117_2724956795/context.md`
- Synthesis: `/Users/matteoscurati/.cache/ai-consultants/consultations/20261021_080117_2724956795/synthesis.json`
- Gemini: `/Users/matteoscurati/.cache/ai-consultants/consultations/20261021_080117_2724956795/gemini.json`
- Codex: `/Users/matteoscurati/.cache/ai-consultants/consultations/20261021_080117_2724956795/codex.json`
- Mistral: `/Users/matteoscurati/.cache/ai-consultants/consultations/20261021_080117_2724956795/mistral.json`
