# DTL Component Patterns — v2.0

## Event Dialect Patterns

### Coordinator

Orchestrates technical operations. One event per operation.

```json
{
  "payloadType": "EVENT",
  "source":      "SigningKeySourceCoordinator",
  "operation":   "WARMUP_FETCH",
  "outcome":     { "phase": "COMPLETED", "result": "SUCCESS" },
  "durationMs":  814,
  "traceId":     "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId":  "f516c69ca51a"
}
```

Required: `source`, `operation`, `outcome.phase`, `outcome.result`, `durationMs`, `instanceId`, `traceId`

---

### Policy

Evaluates domain conditions. `result: UNSATISFIED` is not an error —
it is a legitimate domain state that may trigger downstream action.

```json
{
  "payloadType": "EVENT",
  "source":      "DUPLICATED_ACTIVE_KEY_POLICY",
  "operation":   "CHECK",
  "outcome":     { "phase": "COMPLETED", "result": "CONSISTENT" },
  "durationMs":  0,
  "traceId":     "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId":  "f516c69ca51a"
}
```

Required: `source`, `operation`, `outcome.phase`, `outcome.result`, `durationMs`, `instanceId`, `traceId`

---

### Lifecycle

Records domain entity state transitions.
No `durationMs` — duration belongs to the Coordinators.

```json
{
  "payloadType": "EVENT",
  "source":      "SigningKeyLifecycleManagerImpl",
  "operation":   "CREATE",
  "outcome":     { "phase": "COMPLETED", "result": "SUCCESS" },
  "data": {
    "keyId":          "rsa-1782221123620-3b803784",
    "rotationReason": "expiration"
  },
  "traceId":     "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId":  "f516c69ca51a"
}
```

Required: entity identifier in `data`, `instanceId`, `traceId`


---

### SKIPPED — Infrastructure-Aware Circuit Breaker

Emitted when an operation is consciously not executed because a known
infrastructure state prevented it. The trace remains complete — the skip
is visible, not silent.

```json
{
  "payloadType": "EVENT",
  "source":      "SigningKeySourceCoordinator",
  "operation":   "WARMUP_FETCH",
  "outcome": {
    "phase":  "SKIPPED",
    "result": "INFRA_UNAVAILABLE"
  },
  "durationMs": 0,
  "reason": {
    "component": "REDIS",
    "instance":  "redis-primary-01",
    "state":     "UNAVAILABLE"
  },
  "traceId":    "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId": "f516c69ca51a"
}
```

Required: `source`, `operation`, `outcome.phase: SKIPPED`,
`outcome.result: INFRA_UNAVAILABLE`, `durationMs: 0`, `reason`, `instanceId`, `traceId`

---

## Log Dialect Pattern

Marks the narrative boundaries of a larger operation.

```json
{
  "payloadType": "LOG",
  "resource":    "SigningKeyLifecycleScheduler",
  "message":     "LIFECYCLE_CHECK",
  "state":       "STARTED",
  "traceId":     "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId":  "f516c69ca51a"
}
```

Every LOG must have a `STARTED` / `SUCCESS` (or `FAILED`) pair within the same `traceId`.

---

## Infrastructure Dialect Pattern

See [dtl-infra.md](./dtl-infra.md) for full specification and examples.