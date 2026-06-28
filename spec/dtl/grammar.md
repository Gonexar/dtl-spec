# DTL Grammar — v2.0

## Three Dialects

DTL v2 defines three dialects. Each kind of knowledge has its own grammar.

| Dialect | type | Represents | Grammar |
|---|---|---|---|
| Event | `EVENT` | Domain Fact | Source + Operation + Outcome + Duration |
| Log | `LOG` | Domain Narrative | Narrator + Narrative + State |
| Infrastructure | `INFRA_STATE` | Infrastructure Observation | Component + Probe + Evidence + States + Overall |

---

## Dialect: Event

### Grammar

```
SOURCE + OPERATION + OUTCOME + DURATION
```

Where OUTCOME is composed of:

```
OUTCOME
  ├── PHASE   → execution mechanics  (COMPLETED, FAILED, SKIPPED)
  └── RESULT  → domain semantics     (SUCCESS, CONSISTENT, VALID, EXPIRED, INFRA_UNAVAILABLE)
```

`SKIPPED` signals a consciously bypassed operation — typically because a
DTL-INFRA state reported the required component as `UNAVAILABLE`.
No retry. No timeout. `durationMs` is always `0`.

| Primitive | Field | Rule |
|---|---|---|
| SOURCE | `source` | Stable component name, no dynamic values |
| OPERATION | `operation` | Domain verb in UPPER_SNAKE_CASE |
| OUTCOME | `outcome.phase` + `outcome.result` | `phase`: COMPLETED, FAILED, SKIPPED — `result`: SUCCESS, CONSISTENT, VALID, EXPIRED, UNSATISFIED, INFRA_UNAVAILABLE |
| DURATION | `durationMs` | Integer milliseconds, zero is valid |

---

## Dialect: Log

### Grammar

```
NARRATOR + NARRATIVE + STATE
```

Translated to payload fields:

| Grammar | Payload field | Rationale |
|---|---|---|
| `NARRATOR` | `resource` | The narrator is a system resource |
| `NARRATIVE` | `message` | The narration is a message — universal field name |
| `STATE` | `state` | Retained as-is |

---

## Dialect: Infrastructure

### Grammar

```
COMPONENT + PROBE + EVIDENCE + CAPABILITY STATES + OVERALL STATE
```

See [dtl-infra.md](./dtl-infra.md) for full specification.

---

## Envelope

Common metadata present in every DTL message, regardless of dialect.

```json
{
  "id":        "<uuid-v4>",
  "type":      "EVENT | LOG | INFRA_STATE",
  "level":     "INFO | WARN | ERROR | DEBUG | FATAL",
  "createdAt": "<unix-ms>"
}
```

## Context

Execution correlation metadata, inside every payload.

```json
{
  "traceId":       "<w3c-hex-32>",
  "instanceId":    "<service-instance-id>",
  "tenantId":      "<tenant-id>",
  "correlationId": "<correlation-id>"
}
```

| Field | Required | Format |
|---|---|---|
| `traceId` | Yes | W3C hex 32 chars lowercase |
| `instanceId` | Yes | Service instance identifier |
| `tenantId` | No | Tenant identifier for multi-tenant systems |
| `correlationId` | No | Cross-system correlation identifier |

---

## Naming Convention (Event dialect)

```
<SOURCE>_<RESULT>
<SOURCE>_<OPERATION>_<RESULT>
```

Valid suffixes: `_SUCCESS` `_FAILED` `_ERROR` `_CREATED` `_RETIRING`
`_INACTIVATED` `_VALID` `_EXPIRED` `_CONSISTENT` `_UNSATISFIED` `_SKIPPED`