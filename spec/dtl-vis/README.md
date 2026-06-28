# DTL-VIS — Visual Specification

This directory contains the canonical specification for rendering DTL events as deterministic diagrams.

---

## Files

| File | Contents |
|------|----------|
| [`conformance.md`](./conformance.md) | Ten conformance rules (R1–R10) all renderers must satisfy |
| [`heat-scale.md`](./heat-scale.md) | Seven-tier color scale for duration visualization |
| [`llm-prompt.md`](./llm-prompt.md) | Reference prompt for LLM-driven diagram generation |

---

## Purpose

DTL-VIS defines how a set of DTL events is translated into a visual diagram **deterministically**. Given the same events, any conformant renderer — human, tool, or LLM — produces the same diagram.

---

## Diagram Axes

| Axis | Meaning |
|------|---------|
| **Y-axis** | Time (absolute timestamp + relative offset from trace start) |
| **X-axis** | Six fixed semantic columns (order is mandatory — see R1) |

### Column Order (R1 — mandatory)

```
Orchestration → State → Executions → Cache L1 → Cache L2 → Source / Destination
```

---

## Node Types

| Type | Color | Maps to DTL |
|------|-------|-------------|
| **COMPLETION** | Blue | `EventPayload` with `phase == COMPLETED` |
| **STATE** | Green (success) / Orange (warn) | `EventPayload` with Lifecycle pattern or `InfraStatePayload` |
| **DOMAIN EVENT** | Lilac | `EventPayload` with Policy pattern |

---

## Heat Scale

Duration values are colored by a seven-tier scale (see [`heat-scale.md`](./heat-scale.md)):

| Tier | Range | Color |
|------|-------|-------|
| Instantaneous | 0 ms | `#1B5E20` dark green |
| Very fast | 1–10 ms | `#4CAF50` green |
| Fast | 10–50 ms | `#8BC34A` green-yellow |
| Acceptable | 50–200 ms | `#FFEB3B` yellow |
| Slow | 200 ms–1 s | `#FF9800` orange |
| Very slow | 1 s–2 s | `#F4511E` dark orange |
| Critical | 2 s+ | `#B71C1C` red |

---

## Key Conformance Rules

| Rule | Requirement |
|------|-------------|
| R1 | Column order is fixed (six columns, mandatory left-to-right) |
| R2 | Heat scale uses exactly the seven defined tiers and hex colors |
| R3 | Events sharing `traceId` are always grouped in the same flow |
| R4 | `UNSATISFIED` outcome renders a dashed orange arrow to the next execution |
| R5 | Lifecycle events always appear in the Source/Destination column |
| R6 | Mandatory footer: Legend, Statistics, Performance, Insights |
| R7 | Duration values are colored according to the heat scale |
| R8 | LOG `STARTED` / `SUCCESS` nodes are connected by a vertical dashed line |
| R9 | Multiple traces are separated by labeled horizontal dividers |
| R10 | Heat scale legend appears in the upper right with six reference markers |

Full rule definitions: [`conformance.md`](./conformance.md)

---

## Relationship to DTL Dialects

DTL-VIS renders all three DTL dialects:

| DTL Dialect | Visual representation |
|-------------|----------------------|
| `EVENT` | Node in the appropriate column, colored by outcome and duration |
| `LOG` | Paired STARTED/SUCCESS boundary nodes connected by dashed line |
| `INFRA_STATE` | State node in Orchestration or Executions column reflecting `overall` |

---

## LLM Usage

[`llm-prompt.md`](./llm-prompt.md) provides a self-contained reference prompt for LLM agents generating DTL-VIS conformant diagrams. It summarizes axes, node types, heat scale, conditional arrows, flows, footer, and theme in a single compact block.
