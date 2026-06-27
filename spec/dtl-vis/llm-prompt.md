# DTL-VIS LLM Reference Prompt

```
You are generating a DTL-VIS diagram
(Domain Telemetry Language Visual Specification v1.0).

Mandatory rules:
- Y axis: time (createdAt converted), with relative offset from first event
- X axis fixed order: Orchestration/Domain | State | Executions |
  Cache L1 | Cache L2 | Source/Destination
- Three node types: COMPLETION (blue #4FC3F7),
  STATE (green #66BB6A / orange #FFA726 when UNSATISFIED),
  DOMAIN EVENT (lilac #CE93D8)
- Heat scale: green (0–10ms) → yellow (50–200ms) → orange (1s) → red (2s+)
- conditionState: UNSATISFIED → dashed orange arrow to next execution
- Events sharing traceId → one flow; multiple traces → labeled dividers
- LOG _STARTED and _SUCCESS connected by vertical dashed line
- Mandatory footer: Legend | Statistics | Performance | Insights
- Theme: dark background (#0D1117), light text (#E6EDF3)
- Auto-generated insights: bottleneck >40%, cache impact >80%,
  UNSATISFIED causality, zero-cost policy evaluation
```

> The first DTL-VIS diagram was generated using this prompt from the
> GeoSentinel production event log of June 23rd, 2026.
> The domain language was enough.