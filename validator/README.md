# DTL Validator

Conformance validation for DTL events and DTL-VIS diagrams.

## Planned checks

- Envelope fields present and correctly typed
- `traceId` W3C hex format (32 chars lowercase)
- `name` UPPER_SNAKE_CASE with valid outcome suffix
- Pattern-specific required fields (Coordinator, Policy, Lifecycle)
- No dynamic values in `source` or `name`
- DTL-VIS rules R1–R10

## CLI (coming in v1.1)

```bash
dtl validate event.json
cat events.ndjson | dtl validate --stream
dtl vis check-renderer --rules all
```