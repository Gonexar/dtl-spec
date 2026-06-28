# RFC: Domain Telemetry Specification (DTS)
## Uma Especificação de Telemetria Orientada ao Domínio para a Plataforma GeoSentinel

**Status:** PROPOSED  
**Autor:** Engenharia de Arquitetura GeoSentinel  
**Data:** 24 de Junho de 2026  
**Versão:** v2.0  
**Supersedes:** DTS v1.0  
**Domínio:** Infraestrutura / Observabilidade (OCA)  
**Target:** Observability SDK, Event Store, Consumidores, Ferramentas de Análise  
**License:** CC BY 4.0

---

## 1. Contexto e Motivação

O GeoSentinel possui uma arquitetura de evidências puras (OCA) onde um único
objeto `Event()` é o primitivo central de observabilidade. Todo fato computado
— mutação de domínio, estado de operação, narrativa de lifecycle ou observação
de infraestrutura — trafega pelo mesmo pipeline como bytes opacos, roteados
por headers Kafka sem que a infraestrutura conheça seu conteúdo.

O log de produção capturado em 23/06/2026 demonstra que essa arquitetura
já está funcionando e gerando evidências ricas de domínio. O que a v1.0 não
formalizava era a distinção fundamental entre tipos de conhecimento observável:

- Um **EVENT** representa um fato — algo que aconteceu.
- Um **LOG** representa uma narrativa — a estrutura da história de como algo aconteceu.
- Um **INFRA_STATE** representa uma observação — a condição atual de um componente de infraestrutura.

A DTS v2.0 introduz o modelo de **Dialetos** — cada tipo de conhecimento tem
sua própria gramática, suas próprias perguntas e sua própria representação de
payload. Todos os dialetos compartilham o mesmo envelope e contexto de execução.

---

## 2. Filosofia Fundamental

A DTS é governada por três princípios que não podem ser violados:

**Fato vs. Significado**
O produtor registra o que aconteceu. O consumidor decide o que isso significa.
A infraestrutura não tem opinião sobre nenhum dos dois.

**Domínio como fonte de verdade**
O payload carrega a semântica do domínio intacta. Não é simplificado, achatado
ou normalizado para caber em um modelo genérico de observabilidade. Ferramentas
externas se adaptam ao domínio — não o contrário.

**Interoperabilidade sem subordinação**
O OTel é um protocolo de exportação, não o dono da estrutura. O Event Store
guarda o payload DTS. Consumidores especializados traduzem para OTel quando
necessário, preservando a semântica de domínio em `attributes["dtl.*"]`.

---

## 3. Arquitetura — Três Camadas

Todo mensagem DTL é composta por três camadas:

```
┌─────────────────────────────┐
│          ENVELOPE           │  Metadados comuns — agnóstico ao dialeto
├─────────────────────────────┤
│     PAYLOAD (Dialeto)       │  Gramática específica do conhecimento
├─────────────────────────────┤
│          CONTEXT            │  Correlação de execução
└─────────────────────────────┘
```

### 3.1 Envelope

```json
{
  "id":        "<uuid-v4>",
  "type":      "EVENT | LOG | INFRA_STATE",
  "level":     "INFO | WARN | ERROR | DEBUG | FATAL",
  "createdAt": "<unix-ms>"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | UUID v4 | Sim | Identificador único e imutável — gerado pelo SDK |
| `type` | enum | Sim | Discriminador do dialeto — determina a estrutura do payload |
| `level` | enum | Sim | Severidade para roteamento de infraestrutura — derivado pelo SDK |
| `createdAt` | unix ms | Sim | Clock do produtor no momento do fato — nunca o momento de publicação |

### 3.2 Context

Presente em todo payload, independente do dialeto.

```json
{
  "traceId":       "<w3c-hex-32>",
  "instanceId":    "<service-instance-id>",
  "tenantId":      "<tenant-id>",
  "correlationId": "<correlation-id>"
}
```

| Campo | Obrigatório | Descrição |
|---|---|---|
| `traceId` | Sim | W3C Trace Context hex 32 chars lowercase — fio narrativo entre eventos |
| `instanceId` | Sim | Identificador da instância de serviço |
| `tenantId` | Não | Identificador de tenant para sistemas multi-tenant |
| `correlationId` | Não | Identificador de correlação entre sistemas |

---

## 4. Dialeto: Event (Fato de Domínio)

### 4.1 Gramática

```
SOURCE + OPERATION + OUTCOME + DURATION
```

Onde OUTCOME é composto por duas dimensões:

```
OUTCOME
  ├── PHASE   → mecânica de execução  (COMPLETED, FAILED, SKIPPED)
  └── RESULT  → semântica de domínio  (SUCCESS, CONSISTENT, VALID, EXPIRED,
                                       SATISFIED, UNSATISFIED, INFRA_UNAVAILABLE)
```

`SKIPPED` é um valor de primeira classe — sinaliza que uma operação foi
conscientemente não executada, tipicamente porque um INFRA_STATE reportou
um componente como `UNAVAILABLE`. Sem retry. Sem timeout. `durationMs` é
sempre `0`.

### 4.2 Payload

```json
{
  "payloadType": "EVENT",
  "source":      "VaultCoordinator",
  "operation":   "ENCRYPT",
  "outcome": {
    "phase":  "COMPLETED",
    "result": "SUCCESS"
  },
  "durationMs": 1485,
  "traceId":    "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId": "f516c69ca51a"
}
```

| Campo | Obrigatório | Regra |
|---|---|---|
| `payloadType` | Sim | Sempre `"EVENT"` |
| `source` | Sim | Nome estável do componente. Sem valores dinâmicos. |
| `operation` | Sim | Verbo de domínio em `UPPER_SNAKE_CASE` |
| `outcome.phase` | Sim | `COMPLETED`, `FAILED`, `SKIPPED` |
| `outcome.result` | Sim | `SUCCESS`, `CONSISTENT`, `VALID`, `EXPIRED`, `SATISFIED`, `UNSATISFIED`, `INFRA_UNAVAILABLE` |
| `durationMs` | Sim | Inteiro em milissegundos. Zero é válido. Sempre `0` quando `phase: SKIPPED`. |
| `traceId` | Sim | W3C hex 32 chars |
| `instanceId` | Sim | Identificador da instância |

### 4.3 Padrões de Componentes

#### Coordinator

```json
{
  "payloadType": "EVENT",
  "source":      "SigningKeySourceCoordinator",
  "operation":   "WARMUP_FETCH",
  "outcome":     { "phase": "COMPLETED", "result": "SUCCESS" },
  "durationMs":  814,
  "traceId":     "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId":  "f516c69ca51a"
}
```

#### Policy

`result: UNSATISFIED` não é erro — é um estado legítimo que pode disparar ação downstream.

```json
{
  "payloadType": "EVENT",
  "source":      "DUPLICATED_ACTIVE_KEY_POLICY",
  "operation":   "CHECK",
  "outcome":     { "phase": "COMPLETED", "result": "CONSISTENT" },
  "durationMs":  0,
  "traceId":     "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId":  "f516c69ca51a"
}
```

#### Lifecycle

Registra transições de estado de entidades de domínio. Sem `durationMs` —
duração pertence aos Coordinators.

```json
{
  "payloadType": "EVENT",
  "source":      "SigningKeyLifecycleManagerImpl",
  "operation":   "CREATE",
  "outcome":     { "phase": "COMPLETED", "result": "SUCCESS" },
  "data": {
    "keyId":          "rsa-1782221123620-3b803784",
    "rotationReason": "expiration"
  },
  "traceId":     "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId":  "f516c69ca51a"
}
```

#### SKIPPED — Circuit Breaker orientado a INFRA_STATE

```json
{
  "payloadType": "EVENT",
  "source":      "SigningKeySourceCoordinator",
  "operation":   "WARMUP_FETCH",
  "outcome":     { "phase": "SKIPPED", "result": "INFRA_UNAVAILABLE" },
  "durationMs":  0,
  "reason": {
    "component": "REDIS",
    "instance":  "redis-primary-01",
    "state":     "UNAVAILABLE"
  },
  "traceId":     "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId":  "f516c69ca51a"
}
```

O `reason` aponta para o INFRA_STATE que disparou o skip — fechando o loop
de rastreabilidade entre os dois dialetos. O trace permanece completo e legível.

### 4.4 Convenções de Nomenclatura

```
<SOURCE>_<RESULT>
<SOURCE>_<OPERATION>_<RESULT>

SigningKeySourceCoordinator_SUCCESS
DUPLICATED_ACTIVE_KEY_POLICY_CHECK_CONSISTENT
SIGNING_KEY_CREATED
WARMUP_FETCH_SKIPPED
```

Sufixos válidos: `_SUCCESS` `_FAILED` `_ERROR` `_CREATED` `_RETIRING`
`_INACTIVATED` `_VALID` `_EXPIRED` `_CONSISTENT` `_UNSATISFIED` `_SKIPPED`

---

## 5. Dialeto: Log (Narrativa de Domínio)

### 5.1 Gramática

```
NARRATOR + NARRATIVE + STATE
```

Traduzido para campos de payload com nomenclatura universal:

| Gramática | Campo no payload | Racional |
|---|---|---|
| `NARRATOR` | `resource` | O narrador é um recurso do sistema |
| `NARRATIVE` | `message` | A narração é uma mensagem — nomenclatura universal |
| `STATE` | `state` | Mantido — sem equivalente mais claro |

### 5.2 Payload

```json
{
  "payloadType": "LOG",
  "resource":    "SigningKeyLifecycleScheduler",
  "message":     "LIFECYCLE_CHECK",
  "state":       "STARTED",
  "traceId":     "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId":  "f516c69ca51a"
}
```

```json
{
  "payloadType": "LOG",
  "resource":    "SigningKeyLifecycleScheduler",
  "message":     "LIFECYCLE_CHECK",
  "state":       "SUCCESS",
  "traceId":     "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId":  "f516c69ca51a"
}
```

| Campo | Obrigatório | Regra |
|---|---|---|
| `payloadType` | Sim | Sempre `"LOG"` |
| `resource` | Sim | O recurso do sistema narrando a execução |
| `message` | Sim | A operação sendo narrada. `UPPER_SNAKE_CASE`. |
| `state` | Sim | `STARTED`, `SUCCESS`, `FAILED` |
| `traceId` | Sim | Mesmo `traceId` de todos os EVENTs dessa execução |
| `instanceId` | Sim | Identificador da instância |

### 5.3 Contrato Narrativo

Todo LOG deve ter um par `STARTED` / `SUCCESS` (ou `FAILED`) dentro do mesmo `traceId`:

```
LOG  resource: SigningKeyLifecycleScheduler  message: LIFECYCLE_CHECK  state: STARTED
  [ sequência de EVENTs ]
LOG  resource: SigningKeyLifecycleScheduler  message: LIFECYCLE_CHECK  state: SUCCESS
```

---

## 6. Dialeto: Infrastructure (Observação de Infraestrutura)

### 6.1 Gramática

```
COMPONENT + PROBE + EVIDENCE + CAPABILITY STATES + OVERALL STATE
```

### 6.2 O princípio status/evidence

Cada capacidade observada carrega dois campos:

```
status   → o julgamento traduzido    (opinião — pode ser revisado)
evidence → os dados brutos avaliados (fato — imutável)
```

**O status é uma opinião. O evidence é um fato.**

Persistir os dois juntos habilita reprocessamento retroativo: se um critério de
threshold estava errado, a regra corrigida pode ser aplicada ao evidence
histórico sem re-observar o componente.

### 6.3 Payload

```json
{
  "payloadType": "INFRA_STATE",
  "component":   "REDIS",
  "instance":    "redis-primary-01",
  "probe":       "RedisProbe",
  "evidence": {
    "latency_ms":        4,
    "connected_clients": 142,
    "used_memory_bytes": 847382910
  },
  "states": {
    "latency":    { "status": "NORMAL",    "evidence": { "latency_ms": 4 } },
    "connection": { "status": "ACTIVE",    "evidence": { "connected_clients": 142 } },
    "memory":     { "status": "PRESSURED", "evidence": { "used_memory_bytes": 847382910, "max_memory_bytes": 1073741824 } }
  },
  "overall":    "HEALTHY",
  "traceId":    "6a3a893f847ed2d747f95ea7aaead2c2",
  "instanceId": "f516c69ca51a"
}
```

| `overall` | `level` do envelope | Semântica |
|---|---|---|
| `HEALTHY` | `INFO` | Todas as capacidades dentro dos parâmetros normais |
| `DEGRADED` | `WARN` | Uma ou mais capacidades fora do normal — sistema ainda operacional |
| `UNAVAILABLE` | `ERROR` | Componente inacessível ou em falha crítica |

---

## 7. Resumo dos Dialetos

| Dialeto | type | Representa | Gramática | Campos principais |
|---|---|---|---|---|
| Event | `EVENT` | Fato de Domínio | Source + Operation + Outcome + Duration | `source`, `operation`, `outcome.phase`, `outcome.result`, `durationMs` |
| Log | `LOG` | Narrativa de Domínio | Narrator + Narrative + State | `resource`, `message`, `state` |
| Infrastructure | `INFRA_STATE` | Observação de Infraestrutura | Component + Probe + Evidence + States + Overall | `component`, `probe`, `evidence`, `states`, `overall` |

---

## 8. Interoperabilidade com OpenTelemetry

O payload DTS permanece intacto no Event Store. O OTel é um destino de exportação.

Um consumidor **OTel Exporter** traduz eventos DTS para OTLP preservando a
semântica de domínio em `attributes` com namespace `dtl.*`:

```json
{
  "timestamp":      "2026-06-23T13:25:25.808203761Z",
  "traceId":        "6a3a893f847ed2d747f95ea7aaead2c2",
  "severityNumber": 9,
  "severityText":   "INFO",
  "eventName":      "signing_key.created",
  "attributes": {
    "dtl.source":              "SigningKeyLifecycleManagerImpl",
    "dtl.operation":           "CREATE",
    "dtl.outcome.phase":       "COMPLETED",
    "dtl.outcome.result":      "SUCCESS",
    "dtl.key.id":              "rsa-1782221123620-3b803784",
    "dtl.key.rotation_reason": "expiration",
    "dtl.instance.id":         "f516c69ca51a"
  },
  "resource": {
    "service.name":           "geo-sentinel-gateway",
    "deployment.environment": "production"
  }
}
```

| Campo DTS | Campo OTel |
|---|---|
| `createdAt` | `Timestamp` (ms → nanoseconds) |
| `traceId` | `TraceId` |
| `level` (envelope) | `SeverityNumber` / `SeverityText` |
| `source` | `attributes["dtl.source"]` |
| `operation` | `attributes["dtl.operation"]` |
| `outcome.phase` | `attributes["dtl.outcome.phase"]` |
| `outcome.result` | `attributes["dtl.outcome.result"]` |
| `data.*` | `attributes["dtl.*"]` |
| `id` | `attributes["dtl.event.id"]` |

---

## 9. Impacto no Observability SDK

A API pública não muda. O SDK absorve toda a complexidade da especificação.

```kotlin
observability.event(
  evidence    = SigningKeyCreatedEvidence(keyId = "...", rotationReason = "expiration"),
  persistence = PERSIST,
  dispatch    = DISPATCH
)
```

O SDK garante conformidade DTS automaticamente:

| Responsabilidade | Como |
|---|---|
| Gerar `id` | UUID v4 automático |
| Injetar `createdAt` | Clock do produtor no momento da chamada |
| Injetar `traceId` | Propagado do contexto de execução ativo |
| Injetar `instanceId` | Configuração de bootstrap do serviço |
| Derivar `level` do envelope | A partir de `outcome.result` (Event) ou `overall` (Infrastructure) |
| Validar `source` | Sem valores dinâmicos, nome estável |
| Injetar headers de transporte | `X-Persistence-Policy`, `X-Dispatch-Policy` |
| Serializar em formato DTS canônico | Antes de publicar no Kafka |

---

## 10. O que a DTS não especifica intencionalmente

- O conteúdo de `data` (Event) é livre — pertence ao Schema Registry do SDK
- Thresholds de infraestrutura — o que é `NORMAL` vs `ELEVATED` é definido pelo probe
- Políticas de persistência e dispatch ficam nos headers Kafka, fora do payload
- Interpretação e agregação são responsabilidade dos consumidores

---

## 11. Próximos Passos

1. **Validação retroativa** — verificar que todos os eventos do log de 23/06 estão em conformidade com a estrutura v2. Os eventos observados são majoritariamente alinhados; os campos `status`/`conditionState` precisam ser migrados para `outcome.phase`/`outcome.result`.
2. **SDK** — implementar validação de conformidade v2 em tempo de compilação.
3. **Schema Registry** — formalizar os quatro padrões (Coordinator, Policy, Lifecycle, SKIPPED) como templates de Evidence no SDK.
4. **OTel Exporter** — implementar o consumidor exportador com mapeamento `dtl.*`.
5. **Plataforma DTL-VIS** — renderizador de diagramas a partir de eventos DTS para visualização em tempo real.

---

## 12. Changelog

### v2.0 — 2026-06-24
- Introduzido o modelo de Dialetos: Event, Log, Infrastructure com gramáticas próprias
- Event: `outcome.phase + outcome.result` substitui `status + conditionState`
- Log: gramática `narrator + narrative + state` traduzida para `resource`, `message`, `state`
- Infrastructure promovido a dialeto de primeira classe com `status/evidence` por capacidade
- `SKIPPED` adicionado como valor de PHASE para circuit breaking orientado a INFRA_STATE
- `INFRA_UNAVAILABLE` adicionado como valor de RESULT
- Context formalizado com `tenantId` e `correlationId`
- Namespace OTel atualizado de `dts.*` para `dtl.*`

### v1.0 — 2026-06-24
- Especificação inicial DTS
- Estrutura de envelope + payload (EVENT | LOG)
- Três padrões: Coordinator, Policy, Lifecycle
- Campos de correlação obrigatórios: `traceId`, `instanceId`

---

*GeoSentinel Architecture RFC — Confidential*  
*Licensed under CC BY 4.0 — Gonexar Geographic Intelligence*