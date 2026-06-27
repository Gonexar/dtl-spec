# DTL Spec — Domain Telemetry Language Specification

The open specification for Domain Telemetry Language (DTL) and its visual representation (DTL-VIS).

Created by [Gonexar](https://gonexar.com) — Geographic Intelligence Platform.  
Licensed under [CC BY 4.0](./LICENSE).

---

## What is DTL?

Domain Telemetry Language is a specification for structured observability events where the domain vocabulary is the source of truth. Every event answers four questions:

```
SOURCE + OPERATION + OUTCOME + DURATION = complete domain fact
```

## What is DTL-VIS?

DTL-VIS is the visual specification that defines how DTL events translate into diagrams deterministically. Given the same events, any conformant renderer produces the same diagram.

## Repository Structure

```
dtl-spec/
  spec/          → canonical specifications (DTL + DTL-VIS)
  schemas/       → JSON schemas for event validation
  examples/      → real production event examples
  validator/     → conformance validation rules
```

## Status

| Spec     | Version | Status   |
|----------|---------|----------|
| DTL      | v1.0    | Proposed |
| DTL-VIS  | v1.0    | Proposed |

---

*Reference implementation: [GeoSentinel Observability SDK](https://github.com/gonexar/gonexar-docs)*