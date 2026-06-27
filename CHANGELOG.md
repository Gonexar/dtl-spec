# Changelog

## [2.0.0] — 2026-06-24

### DTL v2.0
- Introduced the Dialect model — Event, Log, Infrastructure as distinct grammars
- Event dialect: `outcome.phase + outcome.result` replaces ambiguous `status + conditionState`
- Log dialect: grammar `narrator + narrative + state` translated to payload fields
  `resource` (narrator), `message` (narrative), `state`
- Infrastructure dialect promoted as first-class (from DTL-INFRA extension)
- Context layer formalized: `traceId`, `instanceId`, `tenantId`, `correlationId`
- `evidence` field added to Infrastructure dialect states
- Status/Evidence separation formalized as design principle

## [1.0.0] — 2026-06-24

### DTL v1.0
- Initial specification of the four grammar primitives: SOURCE, OPERATION, OUTCOME, DURATION
- Canonical event structure: envelope + payload (EVENT | LOG)
- Four component patterns: Coordinator, Policy, Lifecycle, Narrative Log
- Mandatory correlation fields: traceId, instanceId
- Naming conventions: UPPER_SNAKE_CASE, outcome suffixes

### DTL-VIS v1.0
- Six-column canonical layout specification
- Three node types: COMPLETION, STATE, DOMAIN EVENT
- Seven-tier heat scale with hex color definitions
- Ten conformance rules
- LLM reference prompt
- Footer specification: Legend, Statistics, Performance, Insights