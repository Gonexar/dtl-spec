# DTL-INFRA: Infrastructure State Extension
## A DTL Extension for Infrastructure Component Observability

**Status:** PROPOSED  
**Version:** v1.0  
**Parent Spec:** Domain Telemetry Language (DTL) v1.0  
**Date:** 2026-06-24

---

## 1. Overview

DTL-INFRA extends the Domain Telemetry Language to cover infrastructure
component observability. It applies the same philosophy as DTL domain events —
**raw data is translated into meaningful states before being published** —
but targets infrastructure components rather than business domain operations.

Any infrastructure component that can be periodically probed — Redis, Kafka,
PostgreSQL, Elasticsearch, or any other — can be observed through DTL-INFRA.
Consumers receive translated states, never raw metrics.

---

## 2. Philosophy

The same three DTL principles apply without modification:

**InfraProbes produce States** — a probe reads raw metrics from a component
and translates them into a vocabulary of states. It holds the monopoly of
knowledge about what those metrics mean in a given environment.

**INFRA_STATE events preserve Facts** — the translated state is the fact.
It represents the truth about the component at the moment of observation.

**Consumers produce Meaning** — a service that depends on Redis receives
`connection: ACTIVE` and decides what to do with that information. It does
not receive `connected_clients: 142` and compute the meaning itself.

---

## 3. Core Concepts

### 3.1 InfraProbe

An InfraProbe is the agent responsible for:

1. Reading raw metrics from a specific infrastructure component
2. Translating raw metrics into a vocabulary of states
3. Publishing the resulting `INFRA_STATE` event periodically via the
   Observability SDK

Each InfraProbe is specialized for one component type. The translation
criteria — what constitutes `NORMAL` vs `ELEVATED` latency, for example —
are defined by the probe itself, and may vary by environment, team, or SLA.

**This is intentional.** DTL-INFRA does not standardize thresholds.
The vocabulary of states is standardized. The criteria for reaching each
state are not.

### 3.2 Periodic Emission

INFRA_STATE events are emitted periodically, regardless of state changes.
The probe does not wait for a state transition to publish. Every probe
cycle produces one event.

This means consumers always have a recent snapshot of the component's state,
even when everything is healthy. Silence is not a signal — it is an
absence of the probe itself.

### 3.3 Overall State

Every INFRA_STATE event carries an `overall` field that summarizes the
component health in a single value. Consumers that only need a binary
answer read `overall`. Consumers that need granularity read individual
`states` fields.

```
HEALTHY      → all dimensions within normal parameters
DEGRADED     → one or more dimensions outside normal, system still operational
UNAVAILABLE  → component unreachable or critically failed
```

The `overall` value is derived by the probe from individual states.
The derivation logic is defined by the probe — DTL-INFRA does not mandate
a specific aggregation rule.

---

## 4. Event Structure

### 4.1 Envelope

DTL-INFRA events use the standard DTL envelope with `type: INFRA_STATE`:

```json
{
  "id":        "<uuid-v4>",
  "type":      "INFRA_STATE",
  "level":     "INFO | WARN | ERROR",
  "createdAt": "<unix-ms>",
  "payload":   { }
}
```

Envelope `level` is derived from `overall`:

| overall | level |
|---|---|
| `HEALTHY` | `INFO` |
| `DEGRADED` | `WARN` |
| `UNAVAILABLE` | `ERROR` |

### 4.2 Payload

```json
{
  "payloadType": "INFRA_STATE",
  "component":   "<COMPONENT_TYPE>",
  "instance":    "<instance-identifier>",
  "probe":       "<ProbeName>",
  "overall":     "HEALTHY | DEGRADED | UNAVAILABLE",
  "states":      { },
  "instanceId":  "<service-instance-id>",
  "traceId":     "<w3c-hex-32>"
}
```

| Field | Required | Description |
|---|---|---|
| `payloadType` | Yes | Always `"INFRA_STATE"` — discriminator |
| `component` | Yes | Component type in UPPER_SNAKE_CASE: `REDIS`, `KAFKA`, `POSTGRES` |
| `instance` | Yes | Specific instance identifier: `redis-primary-01` |
| `probe` | Yes | Name of the InfraProbe: `RedisProbe`, `KafkaProbe` |
| `overall` | Yes | Aggregated health: `HEALTHY`, `DEGRADED`, `UNAVAILABLE` |
| `states` | Yes | Map of dimension → state. Defined by each probe. |
| `instanceId` | Yes | Identifier of the service instance running the probe |
| `traceId` | Yes | W3C hex 32 chars |

### 4.3 The `states` Map

The `states` field is deliberately open. Each probe defines the dimensions
relevant to its component and the vocabulary of states for each dimension.

There is no global registry of dimensions. There is no mandatory set of
fields. The only constraint is that **all values must be human-readable
state names, never raw metric values.**

```json
// Valid — state is translated, evidence shows what was evaluated
"states": {
  "latency": {
    "status":   "NORMAL",
    "evidence": { "latency_ms": 4 }
  },
  "connection": {
    "status":   "ACTIVE",
    "evidence": { "connected_clients": 142 }
  },
  "memory": {
    "status":   "PRESSURED",
    "evidence": { "used_memory_bytes": 847382910, "max_memory_bytes": 1073741824 }
  }
}

// Invalid — raw metrics as top-level state values
"states": {
  "latency_ms":        4,
  "connected_clients": 142,
  "used_memory_bytes": 847382910
}
```

Each dimension carries two fields:

| Field | Required | Description |
|---|---|---|
| `status` | Yes | The translated state name — always a human-readable string |
| `evidence` | Yes | The raw metrics used to evaluate the status — enables reprocessing, threshold review, and bug investigation |

The `evidence` object is free-form. It contains whatever raw data the probe
read from the component to reach the `status` conclusion. This makes the
evaluate transparent: anyone reading the event can verify why `PRESSURED`
was produced instead of `NORMAL`, without querying the component again.

---

## 5. Naming Convention

Event names follow the pattern:

```
<COMPONENT>_PROBE_<OVERALL>

REDIS_PROBE_HEALTHY
REDIS_PROBE_DEGRADED
REDIS_PROBE_UNAVAILABLE
KAFKA_PROBE_DEGRADED
POSTGRES_PROBE_UNAVAILABLE
```

### 5.1 Recommended Base Vocabulary

DTL-INFRA provides a recommended vocabulary for common dimensions.
Probes may extend or replace it as needed.

| Dimension | States |
|---|---|
| `latency` | `NORMAL` `ELEVATED` `CRITICAL` |
| `connection` | `ACTIVE` `LIMITED` `UNAVAILABLE` |
| `memory` | `NORMAL` `PRESSURED` `CRITICAL` |
| `throughput` | `NORMAL` `REDUCED` `STALLED` |
| `replication` | `SYNCED` `LAGGING` `BROKEN` |
| `availability` | `ONLINE` `PARTIAL` `OFFLINE` |

Component-specific dimensions are encouraged and expected. A Kafka probe
adds `consumer_lag`. A Postgres probe adds `replication_slot`. A Redis
probe adds `eviction`. These are defined by whoever builds the probe —
not by this specification.

---

## 6. Examples

### Redis — Healthy

```json
{
  "id": "a3f2b1c4-9d8e-4f7a-b6c5-1e2d3f4a5b6c",
  "type": "INFRA_STATE", "level": "INFO",
  "createdAt": 1782221125613,
  "payload": {
    "payloadType": "INFRA_STATE",
    "component":   "REDIS",
    "instance":    "redis-primary-01",
    "probe":       "RedisProbe",
    "overall":     "HEALTHY",
    "states": {
      "latency":    { "status": "NORMAL", "evidence": { "latency_ms": 4 } },
      "connection": { "status": "ACTIVE", "evidence": { "connected_clients": 142 } },
      "memory":     { "status": "NORMAL", "evidence": { "used_memory_bytes": 214748364, "max_memory_bytes": 1073741824 } },
      "eviction":   { "status": "CLEAN",  "evidence": { "evicted_keys": 0 } }
    },
    "instanceId": "f516c69ca51a",
    "traceId":    "6a3a893f847ed2d747f95ea7aaead2c2"
  }
}
```

### Kafka — Degraded

```json
{
  "id": "b4c3d2e1-8f7a-4b6c-a5d4-2f3e4d5c6b7a",
  "type": "INFRA_STATE", "level": "WARN",
  "createdAt": 1782221185613,
  "payload": {
    "payloadType": "INFRA_STATE",
    "component":   "KAFKA",
    "instance":    "kafka-broker-03",
    "probe":       "KafkaProbe",
    "overall":     "DEGRADED",
    "states": {
      "connection":   { "status": "ACTIVE",   "evidence": { "active_connections": 89 } },
      "throughput":   { "status": "NORMAL",   "evidence": { "messages_per_sec": 12400 } },
      "consumer_lag": { "status": "ELEVATED", "evidence": { "lag_records": 84200, "lag_ms": 4300 } },
      "replication":  { "status": "LAGGING",  "evidence": { "replication_lag_ms": 1850 } }
    },
    "instanceId": "f516c69ca51a",
    "traceId":    "7b4b994g958fe3e858g06fb8bbfbe3d3"
  }
}
```

### PostgreSQL — Unavailable

```json
{
  "id": "c5d4e3f2-7a6b-4c5d-b4e3-3a4b5c6d7e8f",
  "type": "INFRA_STATE", "level": "ERROR",
  "createdAt": 1782221245613,
  "payload": {
    "payloadType": "INFRA_STATE",
    "component":   "POSTGRES",
    "instance":    "postgres-replica-02",
    "probe":       "PostgresProbe",
    "overall":     "UNAVAILABLE",
    "states": {
      "connection":  { "status": "UNAVAILABLE", "evidence": { "error": "connection refused", "attempts": 3 } },
      "replication": { "status": "BROKEN",      "evidence": { "replication_slot_active": false, "lag_bytes": -1 } }
    },
    "instanceId": "f516c69ca51a",
    "traceId":    "8c5c005h069gf4f969h17gc9ccgcf4e4"
  }
}
```

---

## 7. SDK Integration

DTL-INFRA events are published through the same Observability SDK as
domain events. The producer interface is identical:

```kotlin
observability.event(
  evidence = RedisProbeEvidence(
    instance = "redis-primary-01",
    overall  = Overall.HEALTHY,
    states   = mapOf(
      "latency"    to DimensionState("NORMAL", mapOf("latency_ms" to 4)),
      "connection" to "ACTIVE",
      "memory"     to "NORMAL",
      "eviction"   to "CLEAN"
    )
  ),
  persistence = PERSIST,
  dispatch    = DISPATCH
)
```

### Recommended transport policies

| Scenario | Persistence | Dispatch | Rationale |
|---|---|---|---|
| `HEALTHY` (routine) | `PERSIST` | `NONE` | Stored for history, no real-time noise |
| `DEGRADED` | `PERSIST` | `DISPATCH` | Consumers need to know now |
| `UNAVAILABLE` | `PERSIST` | `DISPATCH` | Critical — emit immediately |

---

## 8. What DTL-INFRA Does Not Specify

**Probe implementation** — how a probe connects to a component, which
client library it uses, and how frequently it polls are implementation
details left to each probe author.

**Threshold criteria** — what constitutes `ELEVATED` latency for Redis
in production may differ from staging, from a high-throughput service, or
from a latency-sensitive one. DTL-INFRA intentionally leaves threshold
definition to the probe and its maintainers.

**Dimension completeness** — there is no required set of dimensions for
any component type. A minimal Redis probe may only report `connection`
and `overall`. A comprehensive one may report ten dimensions. Both are
valid DTL-INFRA.

**Aggregation rules for `overall`** — how a probe derives `overall` from
individual states is its own business logic. DTL-INFRA only mandates
that `overall` exists and uses the three canonical values.

---

## 9. Relationship to DTL Domain Events

DTL-INFRA is an extension of DTL, not a separate specification. It shares:

- The same envelope structure
- The same Observability SDK
- The same Kafka transport pipeline
- The same Event Store
- The same `traceId` correlation model

The only differences are the `type: INFRA_STATE` discriminator and the
open `states` map — which replaces the four mandatory DTL grammar
primitives with a flexible, probe-defined vocabulary.

A DTL-VIS renderer that understands `INFRA_STATE` events can visualize
infrastructure health alongside domain operations in the same diagram —
because they speak the same language.

---

*DTL-INFRA v1.0 — Part of the Domain Telemetry Language Specification*  
*Licensed under CC BY 4.0 — Gonexar Geographic Intelligence*