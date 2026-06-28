# RFCs — Request for Comments

Design documents that record the motivation, decisions, and rationale behind DTL specifications.

---

## Index

| RFC | Title | Status | Supersedes |
|-----|-------|--------|------------|
| [RFC-0001](./RFC-0001-domain-telemetry-language.md) | Domain Telemetry Language v2.0 | PROPOSED | DTS v1.0 |
| [RFC-0002](./RFC-0002-dtl-vis-visual-specification.md) | DTL-VIS Visual Specification v1.0 | PROPOSED | — |
| [RFC-0003](./RFC-0003-DTL-INFRA.md) | DTL-INFRA Infrastructure State Extension v1.0 | PROPOSED | — |

---

## RFC-0001 — Domain Telemetry Language v2.0

Defines the full DTL v2.0 specification:

- Three-dialect model: `EVENT`, `LOG`, `INFRA_STATE`
- `DtlEnvelope<T>` common wrapper with derived `DtlLevel`
- `DtlContext` shared across all dialects (traceId, instanceId, tenantId, correlationId)
- `Outcome` with two orthogonal dimensions: `phase` (HOW) and `result` (WHAT)
- Four component patterns: Coordinator, Policy, Lifecycle, Circuit Breaker
- OTel export mapping via `attributes["dtl.*"]` namespace
- Three core principles: Fact vs. Meaning, Domain as source of truth, Interoperability without subordination

**Key change from v1.0:** Adds the `LOG` and `INFRA_STATE` dialects. Separates outcome into `phase` + `result`. Formalizes `SKIPPED` as a first-class phase with mandatory `SkipReason`.

---

## RFC-0002 — DTL-VIS Visual Specification v1.0

Defines the deterministic visual rendering of DTL events:

- Six-column fixed layout (Orchestration → State → Executions → Cache L1 → Cache L2 → Source/Destination)
- Three node types: COMPLETION, STATE, DOMAIN EVENT
- Seven-tier heat scale with exact hex color codes
- Ten conformance rules (R1–R10) for renderer implementers
- LLM-agent reference prompt for diagram generation

---

## RFC-0003 — DTL-INFRA Infrastructure State Extension v1.0

Defines the `INFRA_STATE` dialect in detail:

- InfraProbe contract: periodic observation regardless of state changes
- `CapabilityState` structure: `status` (opinion) + `evidence` (immutable raw metrics)
- `OverallState` aggregation: `HEALTHY`, `DEGRADED`, `UNAVAILABLE`
- Status vs. evidence separation: enables retroactive rule reprocessing on historical data
- Transport policy defaults: `HEALTHY → NONE`, `DEGRADED/UNAVAILABLE → DISPATCH`
- Recommended status vocabulary: Latency, Connection, Memory, Throughput, Replication, Eviction, Availability

---

## Conventions

- RFCs are written primarily in Portuguese with English annotations for grammar and type definitions.
- A new RFC is required for any change to the DTL grammar, envelope structure, or DTL-VIS conformance rules.
- Breaking changes must increment the spec version and update the Status table in the root README.
