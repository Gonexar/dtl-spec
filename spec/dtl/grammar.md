# DTL Grammar

## The Four Primitives

Every DTL EVENT must express:

```
SOURCE + OPERATION + OUTCOME + DURATION
```

| Primitive | Field | Rule |
|---|---|---|
| SOURCE | `payload.source` | Stable component name, no dynamic values |
| OPERATION | `data.operation` | Domain verb in UPPER_SNAKE_CASE |
| OUTCOME | `data.status` or `data.conditionState` | See outcome vocabulary |
| DURATION | `data.executionTimeMs` | Integer milliseconds, zero is valid |

## Envelope

```json
{
  "id":        "<uuid-v4>",
  "type":      "EVENT | LOG",
  "level":     "INFO | WARN | ERROR | DEBUG | FATAL",
  "createdAt": "<unix-ms>",
  "payload":   {}
}
```

## Naming Convention

```
<SOURCE>_<OUTCOME>
<SOURCE>_<OPERATION>_<OUTCOME>
```

Valid suffixes: `_SUCCESS` `_FAILED` `_ERROR` `_CREATED`
`_RETIRING` `_INACTIVATED` `_VALID` `_EXPIRED` `_CONSISTENT`

## Mandatory Correlation Fields

| Field | Type | Format |
|---|---|---|
| `traceId` | string | W3C hex 32 chars lowercase |
| `instanceId` | string | Service instance identifier |