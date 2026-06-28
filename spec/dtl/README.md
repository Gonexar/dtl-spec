# DTL — Domain Telemetry Language Specification

This directory contains the canonical grammar and patterns for DTL v2.0.

---

## Files

| File | Contents |
|------|----------|
| [`grammar.md`](./grammar.md) | Three-dialect grammar, envelope structure, context fields, outcome semantics |
| [`patterns.md`](./patterns.md) | Four component patterns: Coordinator, Policy, Lifecycle, Circuit Breaker |

---

## Three Dialects

DTL v2.0 defines three dialects, each represented by a distinct `DtlType`:

### EVENT — Domain Facts

Answers: **Who acted → What did they do → How did it finish → What was the result → How long did it take**

```
SOURCE + OPERATION + OUTCOME + DURATION = complete domain fact
```

```json
{
  "type": "EVENT",
  "payload": {
    "source": "SigningKeySourceCoordinator",
    "operation": "WARMUP_FETCH",
    "outcome": { "phase": "COMPLETED", "result": "SUCCESS" },
    "durationMs": 142,
    "data": { "keyCount": 3 },
    "context": { "traceId": "...", "instanceId": "..." }
  }
}
```

### LOG — Domain Narratives

Answers: **Who is narrating → Which narrative → What state is the narrative in**

```
NARRATOR + NARRATIVE + STATE = domain narrative boundary
```

Every narrative **must** have a `STARTED` / `SUCCESS` (or `FAILED`) pair sharing the same `traceId`.

```json
{
  "type": "LOG",
  "payload": {
    "resource": "SigningKeyLifecycleManagerImpl",
    "message": "LIFECYCLE_CHECK",
    "state": "STARTED",
    "context": { "traceId": "...", "instanceId": "..." }
  }
}
```

### INFRA_STATE — Component Observations

Answers: **Which component → Which probe → What evidence → What is each capability's state → What is the overall health**

```
COMPONENT + PROBE + EVIDENCE + STATES + OVERALL = component health snapshot
```

Emitted periodically by InfraProbes regardless of state changes.

```json
{
  "type": "INFRA_STATE",
  "payload": {
    "component": "REDIS",
    "instance": "redis-primary-01",
    "probe": "RedisHealthProbe",
    "evidence": { "latencyMs": 2.1, "connectedClients": 18 },
    "states": {
      "latency": { "status": "NORMAL", "evidence": { "latencyMs": 2.1 } },
      "connections": { "status": "ACTIVE", "evidence": { "connectedClients": 18 } }
    },
    "overall": "HEALTHY",
    "context": { "traceId": "...", "instanceId": "..." }
  }
}
```

---

## Shared Envelope

All three dialects share the same envelope wrapper:

```json
{
  "id": "<uuid-v4>",
  "type": "EVENT | LOG | INFRA_STATE",
  "level": "DEBUG | INFO | WARN | ERROR | FATAL",
  "createdAt": 1719100800000
}
```

`level` is always **derived** by the SDK — producers never set it manually.

| Dialect | Condition | Level |
|---------|-----------|-------|
| EVENT | phase COMPLETED, result SUCCESS/CONSISTENT/SATISFIED/VALID | INFO |
| EVENT | phase COMPLETED, result UNSATISFIED/EXPIRED/INCONSISTENT | WARN |
| EVENT | phase FAILED | ERROR |
| EVENT | phase SKIPPED | WARN |
| LOG | state STARTED or SUCCESS | INFO |
| LOG | state FAILED | ERROR |
| INFRA_STATE | overall HEALTHY | INFO |
| INFRA_STATE | overall DEGRADED | WARN |
| INFRA_STATE | overall UNAVAILABLE | ERROR |

---

## Shared Context

`DtlContext` is embedded in every payload:

| Field | Required | Format | Purpose |
|-------|----------|--------|---------|
| `traceId` | yes | 32-char lowercase hex (W3C) | Correlates all events in a flow |
| `instanceId` | yes | string | Identifies the service instance |
| `tenantId` | no | string | Multi-tenant isolation |
| `correlationId` | no | string | Cross-system tracing |

---

## Core Principles

**1. Fact vs. Meaning** — The producer records what happened. The consumer decides what it means. A `UNSATISFIED` result is not an error; it is a domain fact.

**2. Domain as source of truth** — Payload carries domain semantics intact. No information is simplified to fit a generic observability model.

**3. Interoperability without subordination** — OpenTelemetry is an export protocol, not the owner of the structure. DTL semantics live in `attributes["dtl.*"]` when mapped to OTel.

**4. Status vs. Evidence** — In `INFRA_STATE`, status is an opinion (may be revised); evidence is a fact (immutable raw metrics). This enables retroactive rule reprocessing.
