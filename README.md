# DTL Spec — Domain Telemetry Language Specification

The open specification for Domain Telemetry Language (DTL) and its visual representation (DTL-VIS).

Created by [Gonexar](https://gonexar.com) — Geographic Intelligence Platform.  
Licensed under [CC BY 4.0](./LICENSE).

---

## What is DTL?

Domain Telemetry Language is a specification for structured observability events where the domain vocabulary is the source of truth. DTL defines three dialects, each answering a different observability question:

| Dialect | Type | Question answered |
|---------|------|-------------------|
| **Event** | `EVENT` | What domain fact occurred? |
| **Log** | `LOG` | What narrative boundary was crossed? |
| **Infrastructure** | `INFRA_STATE` | What is the health of a component? |

Every **Event** answers four questions:

```
SOURCE + OPERATION + OUTCOME + DURATION = complete domain fact
```

Every **Log** answers three questions:

```
NARRATOR + NARRATIVE + STATE = domain narrative boundary
```

Every **Infrastructure** observation answers five questions:

```
COMPONENT + PROBE + EVIDENCE + STATES + OVERALL = component health snapshot
```

## What is DTL-VIS?

DTL-VIS is the visual specification that defines how DTL events translate into diagrams deterministically. Given the same events, any conformant renderer produces the same diagram.

## Repository Structure

```
dtl-spec/
  spec/
    dtl/         → DTL grammar, three dialects, outcome semantics, component patterns
    dtl-vis/     → Visual rendering spec, conformance rules, heat scale, LLM prompt
  schemas/       → JSON Schema validation for events and patterns
  rfcs/          → Design documents (RFC-0001 DTL, RFC-0002 DTL-VIS, RFC-0003 DTL-INFRA)
  examples/      → Real production event examples
  validator/     → Conformance validation rules (CLI coming in v1.1)
```

## DTL Core Type Hierarchy

The reference implementation (`dtl-core`) expresses the full type system as pure Kotlin with zero external dependencies:

```
DtlEnvelope<T : DtlPayload>
├── id: UUID v4
├── type: DtlType (EVENT | LOG | INFRA_STATE)
├── level: DtlLevel (derived — DEBUG | INFO | WARN | ERROR | FATAL)
├── createdAt: Long (producer clock, milliseconds)
└── payload: T
    ├── EventPayload       — domain facts
    ├── LogPayload         — narrative boundaries
    └── InfraStatePayload  — component observations
```

`DtlContext` is shared across all three dialects:

```
DtlContext
├── traceId: String       (W3C Trace Context, 32-char lowercase hex — mandatory)
├── instanceId: String    (service instance — mandatory)
├── tenantId: String?     (multi-tenant — optional)
└── correlationId: String? (cross-system correlation — optional)
```

## Outcome Semantics

Outcome has two orthogonal dimensions:

| `phase` (HOW) | `result` (WHAT) | Meaning | `DtlLevel` |
|---|---|---|---|
| `COMPLETED` | `SUCCESS` | Operation succeeded | INFO |
| `COMPLETED` | `CONSISTENT` | Policy found system consistent | INFO |
| `COMPLETED` | `SATISFIED` | Policy condition was met | INFO |
| `COMPLETED` | `UNSATISFIED` | Policy condition not met — valid domain state | WARN |
| `COMPLETED` | `EXPIRED` | Entity TTL elapsed | WARN |
| `FAILED` | `FAILED` | Operation failed | ERROR |
| `SKIPPED` | `INFRA_UNAVAILABLE` | Circuit breaker — operation bypassed | WARN |

> `UNSATISFIED` is not an error. It is a legitimate domain outcome signaling that a policy's condition was evaluated and not met.

## Component Patterns

Four structural patterns classify how a producer uses an `EventPayload`:

| Pattern | Signal | Key constraint |
|---------|--------|----------------|
| **Coordinator** | `phase == COMPLETED && durationMs > 0` | Records operation with execution time |
| **Policy** | `result in {SATISFIED, UNSATISFIED, CONSISTENT, INCONSISTENT}` | Evaluates domain condition |
| **Lifecycle** | Entity id present in `data` | Records entity state transition; no `durationMs` |
| **Circuit Breaker** | `phase == SKIPPED` | Operation consciously not executed; `durationMs == 0`, `reason` required |

## Status

| Spec | Version | Status |
|------|---------|--------|
| DTL v2.0 | `EVENT` + `LOG` dialects | Proposed |
| DTL-INFRA v1.0 | `INFRA_STATE` dialect | Proposed |
| DTL-VIS v1.0 | Visual rendering | Proposed |

---

*Reference implementation: [gonexar-oca-sdk / dtl-core](https://github.com/gonexar/gonexar-oca-sdk)*
