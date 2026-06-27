# DTL Component Patterns

## Coordinator
One event per operation. Required: `operation`, `status`, `executionTimeMs`, `instanceId`, `traceId`

## Policy
Evaluates conditions. `UNSATISFIED` is not an error.
Required: `operation`, `status`, `conditionState`, `executionTimeMs`, `instanceId`, `traceId`

## Lifecycle
Records entity state transitions. No `executionTimeMs`.
Required: entity identifier, `instanceId`, `traceId`

## Narrative Log
Marks operation boundaries. Every LOG must have `_STARTED` / `_SUCCESS` pair within same `traceId`.