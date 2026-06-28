# DTL Validator

Conformance validation for DTL events and DTL-VIS diagrams.

---

## Planned Checks

### Envelope

- `id` is a valid UUID v4
- `type` is one of `EVENT`, `LOG`, `INFRA_STATE`
- `level` is one of `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`
- `createdAt` is a positive integer (milliseconds)

### Context (all dialects)

- `traceId` matches W3C format: 32-char lowercase hex `^[a-f0-9]{32}$`
- `instanceId` is present and non-blank
- `tenantId` and `correlationId` are strings when present

### EVENT dialect

- `source` is non-blank and contains no dynamic values (no UUIDs, timestamps, or IDs embedded)
- `operation` is `UPPER_SNAKE_CASE`
- `outcome.phase` is one of `COMPLETED`, `FAILED`, `SKIPPED`
- `outcome.result` is a valid `OutcomeResult` value
- `durationMs >= 0`
- When `phase == SKIPPED`: `durationMs == 0` and `reason` is present
- When `phase == SKIPPED`: `result == INFRA_UNAVAILABLE`
- `reason.component` matches an `INFRA_STATE` event component in the same trace

### LOG dialect

- `resource` is non-blank
- `message` is `UPPER_SNAKE_CASE`
- `state` is one of `STARTED`, `SUCCESS`, `FAILED`
- Every `STARTED` log has a matching `SUCCESS` or `FAILED` within the same `traceId`

### INFRA_STATE dialect

- `component` is `UPPER_SNAKE_CASE` and non-blank
- `instance` and `probe` are non-blank
- `states` map has at least one entry
- Each `CapabilityState.status` is `UPPER_SNAKE_CASE`
- `overall` is one of `HEALTHY`, `DEGRADED`, `UNAVAILABLE`
- `level` matches the derived value for `overall` (HEALTHY→INFO, DEGRADED→WARN, UNAVAILABLE→ERROR)

### DTL-VIS Conformance

- Rules R1–R10 from `spec/dtl-vis/conformance.md` verified against a rendered diagram
- Column order is fixed and matches specification
- Heat scale colors match the seven defined hex values
- Events sharing `traceId` are grouped in the same flow
- Footer sections (Legend, Statistics, Performance, Insights) are present

---

## CLI (coming in v1.1)

```bash
dtl validate event.json
cat events.ndjson | dtl validate --stream
dtl vis check-renderer --rules all
```

---

## Schema Validation

JSON Schema files in `../schemas/` can be used for immediate structural validation before running semantic checks:

| Schema | Validates |
|--------|-----------|
| [`schemas/events/event.schema.json`](../schemas/events/event.schema.json) | DTL event envelope (all dialects) |
| [`schemas/patterns/coordinator.schema.json`](../schemas/patterns/coordinator.schema.json) | Coordinator pattern payload |
