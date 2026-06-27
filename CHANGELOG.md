# Changelog

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