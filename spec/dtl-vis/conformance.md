# DTL-VIS Conformance Rules

Conformance is binary — there is no partial conformance.

| # | Rule |
|---|---|
| R1 | Column order: Orchestration → State → Executions → Cache L1 → Cache L2 → Source/Destination |
| R2 | Heat scale uses the seven defined tiers and colors |
| R3 | Events sharing `traceId` always grouped in the same flow |
| R4 | `conditionState: UNSATISFIED` → dashed orange arrow to next execution node |
| R5 | Domain Events (Lifecycle) always in Source/Destination column |
| R6 | Mandatory footer: Legend, Statistics, Performance, Insights |
| R7 | Duration values colored by heat scale |
| R8 | LOG `_STARTED` and `_SUCCESS` connected by vertical dashed line |
| R9 | Multiple traces separated by labeled horizontal dividers |
| R10 | Heat scale legend in upper right with six reference markers |