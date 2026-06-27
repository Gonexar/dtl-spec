# RFC: Domain Telemetry Specification (DTS)
## Uma Especificação de Telemetria Orientada ao Domínio para a Plataforma GeoSentinel

**Status:** PROPOSED  
**Autor:** Engenharia de Arquitetura GeoSentinel  
**Data:** 24 de Junho de 2026  
**Versão:** v1.0  
**Domínio:** Infraestrutura / Observabilidade (OCA)  
**Target:** Observability SDK, Event Store, Consumidores, Ferramentas de Análise

---

## 1. Contexto e Motivação

O GeoSentinel possui uma arquitetura de evidências puras (OCA) onde um único objeto `Event()` é o primitivo central de observabilidade. Todo fato computado — mutação de domínio, estado de operação, log de lifecycle ou métrica de performance — trafega pelo mesmo pipeline como bytes opacos, roteados por headers Kafka sem que a infraestrutura conheça seu conteúdo.

O log de produção capturado em 23/06/2026 demonstra que essa arquitetura já está funcionando e gerando evidências ricas de domínio:

```json
[{
  "id": "05dd6394-5d56-4243-874a-71344760bfad",
  "type": "EVENT",
  "level": "INFO",
  "payload": {
    "payloadType": "EVENT",
    "source": "SigningKeyLifecycleManagerImpl",
    "name": "SIGNING_KEY_CREATED",
    "data": {
      "keyId": "rsa-1782221123620-3b803784",
      "rotationReason": "expiration",
      "instanceId": "f516c69ca51a",
      "traceId": "6a3a893f847ed2d747f95ea7aaead2c2"
    }
  },
  "createdAt": 1782221125613
},
{
  "id": "8df8b5ff-a4af-426f-b4bd-e57cfb882bce",
  "type": "LOG",
  "level": "INFO",
  "payload": {
    "payloadType": "LOG",
    "source": "SigningKeyLifecycleScheduler",
    "level": "LIFECYCLE_CHECK_STARTED",
    "message": "Starting signing key lifecycle evaluation.",
    "fields": {
      "timestamp_utc": "2026-06-23T13:25:21Z",
      "instanceId": "f516c69ca51a",
      "traceId": "6a3a893f847ed2d747f95ea7aaead2c2"
    }
  },
  "createdAt": 1782221121603
}]
```

Esses dois registros são estados do **mesmo objeto `Event()`** — um carrega um fato de domínio, outro carrega uma mensagem de diagnóstico. Ambos possuem semântica rica, rastreabilidade por `traceId`, correlação temporal e contexto de instância.

O que está faltando é uma **especificação formal** que:

1. Canonize a estrutura atual como um padrão de mercado próprio.
2. Incorpore diretrizes de interoperabilidade do OpenTelemetry sem sacrificar a semântica de domínio.
3. Defina contratos claros para produtores, SDK e consumidores.

Esta RFC propõe a **Domain Telemetry Specification (DTS)** — um padrão de telemetria orientado ao domínio, onde a semântica é de quem produz e de quem consome, e a infraestrutura permanece cega.

---

## 2. Filosofia Fundamental

A DTS é governada por três princípios que não podem ser violados:

**Fato vs. Significado**
O produtor registra o que aconteceu. O consumidor decide o que isso significa. A infraestrutura não tem opinião sobre nenhum dos dois.

**Domínio como fonte de verdade**
O payload carrega a semântica do domínio intacta. Não é simplificado, achatado ou normalizado para caber em um modelo genérico de observabilidade. Ferramentas externas se adaptam ao domínio — não o contrário.

**Interoperabilidade sem subordinação**
A DTS adota campos e convenções do OpenTelemetry onde fazem sentido — especialmente `traceId`, `spanId` e `level`. Mas o OTel é um protocolo de exportação, não o dono da estrutura. O Event Store guarda o payload DTS. Consumidores especializados traduzem para OTel quando necessário.

---

## 3. Estrutura Canônica do Domain Telemetry Event

### 3.1 Envelope (campos obrigatórios)

```json
{
  "id":        "<uuid-v4>",
  "type":      "<EVENT | LOG>",
  "level":     "<INFO | WARN | ERROR | DEBUG | FATAL>",
  "createdAt": "<unix-ms>",
  "payload":   { ... }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | UUID v4 | Sim | Identificador único e imutável da evidência |
| `type` | enum | Sim | Natureza da evidência: `EVENT` (fato de domínio) ou `LOG` (mensagem de diagnóstico) |
| `level` | enum | Sim | Severidade: `INFO`, `WARN`, `ERROR`, `DEBUG`, `FATAL` |
| `createdAt` | unix milissegundos | Sim | Timestamp de criação pela origem — clock do produtor |

> **Diretriz OTel incorporada:** `createdAt` mapeia ao campo `Timestamp` do OTel LogRecord (quando da ocorrência, não quando observado pelo pipeline). O SDK pode injetar `observedAt` como campo adicional no envelope para rastrear quando o evento entrou na esteira.

### 3.2 Payload de EVENT (fato de domínio)

```json
{
  "payloadType": "EVENT",
  "source":      "<ComponenteOuPolitica>",
  "name":        "<NOME_DO_EVENTO_EM_UPPERCASE_SNAKE>",
  "data":        {  }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `payloadType` | enum | Sim | Discriminador do tipo de payload: `EVENT` |
| `source` | string | Sim | Componente, coordenador ou política que gerou o fato |
| `name` | string | Sim | Nome do evento em `UPPER_SNAKE_CASE`. Identifica unicamente a estrutura do `data` |
| `data` | object | Sim | Estado computado do domínio — livre, rico, estruturado conforme o contrato do evento |

**Exemplo real do log de produção:**

```json
{
  "payloadType": "EVENT",
  "source":      "SigningKeyLifecycleManagerImpl",
  "name":        "SIGNING_KEY_CREATED",
  "data": {
    "keyId":          "rsa-1782221123620-3b803784-426e-4cd9-8497-a9a76b5459dc",
    "rotationReason": "expiration",
    "instanceId":     "f516c69ca51a",
    "traceId":        "6a3a893f847ed2d747f95ea7aaead2c2"
  }
}
```

### 3.3 Payload de LOG (mensagem de diagnóstico)

```json
{
  "payloadType": "LOG",
  "source":      "<ComponenteOuClasse>",
  "level":       "<NIVEL_SEMANTICO_DO_DOMINIO>",
  "message":     "<mensagem humana legível>",
  "fields":      { }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `payloadType` | enum | Sim | Discriminador do tipo de payload: `LOG` |
| `source` | string | Sim | Componente ou classe que emitiu o log |
| `level` | string | Sim | Nível semântico do domínio (ex: `LIFECYCLE_CHECK_STARTED`) — não necessariamente igual ao `level` do envelope |
| `message` | string | Sim | Mensagem legível por humanos |
| `fields` | object | Sim | Contexto estruturado — livre, inclui `traceId`, `instanceId`, timestamps |

**Exemplo real do log de produção:**

```json
{
  "payloadType": "LOG",
  "source":      "SigningKeyLifecycleScheduler",
  "level":       "LIFECYCLE_CHECK_SUCCESS",
  "message":     "Lifecycle evaluation completed successfully.",
  "fields": {
    "timestamp_utc": "2026-06-23T13:25:25.808203761Z",
    "instanceId":    "f516c69ca51a",
    "traceId":       "6a3a893f847ed2d747f95ea7aaead2c2"
  }
}
```

---

## 4. Campos de Correlação e Rastreabilidade

O log de produção demonstra que `traceId` já está presente em praticamente todos os eventos e logs. A DTS formaliza isso como **campo obrigatório de correlação**, alinhado às especificações W3C Trace Context adotadas pelo OpenTelemetry.

### 4.1 Campos de correlação obrigatórios dentro de `data` (EVENT) e `fields` (LOG)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `traceId` | string hex 32 chars | Sim | ID do trace distribuído — propaga o contexto de execução entre componentes |
| `instanceId` | string | Sim | Identificador da instância do serviço que gerou o evento |

### 4.2 Campos de correlação recomendados

| Campo | Tipo | Descrição |
|---|---|---|
| `spanId` | string hex 16 chars | ID do span dentro do trace — permite correlação com traces OTel |
| `app` | string | Nome da aplicação (ex: `geo-sentinel-gateway`) |
| `executionTimeMs` | int | Tempo de execução da operação em milissegundos |
| `operation` | string | Nome da operação executada (ex: `SIGN_JWT`, `ENCRYPT`) |
| `status` | string | Estado resultante da operação (ex: `COMPLETED`, `VALID`, `CONSISTENT`) |

> **Diretriz OTel incorporada:** `traceId` e `spanId` seguem o formato W3C Trace Context (hex lowercase), compatível com qualquer backend OTel. Isso permite que consumidores exportadores correlacionem eventos DTS com traces distribuídos sem transformação de formato.

---

## 5. Convenções de Nomenclatura

### 5.1 Nome do EVENT (`name`)

O `name` identifica unicamente a estrutura do `data`. É o contrato do evento.

**Formato:** `<SOURCE>_<RESULTADO>` ou `<SOURCE>_<OPERACAO>_<RESULTADO>`

```
SIGNING_KEY_CREATED
SIGNING_KEY_RETIRING
SIGNING_KEY_INACTIVATED
DUPLICATED_ACTIVE_KEY_POLICY_CHECK_CONSISTENT
EXPIRATION_TIME_ROTATION_CHECK_ROTATION_ACTIVE_KEY_VALID
API_KEY_VALIDATOR_VALIDATE_VALID
TOKEN_SIGNER_COORDINATOR_SUCCESS
```

**Regras:**
- `UPPER_SNAKE_CASE` obrigatório
- Não deve conter valores dinâmicos (IDs, timestamps) — esses vão em `data`
- Deve ser estável — uma mudança no `name` é uma breaking change de contrato
- O sufixo indica o resultado: `_SUCCESS`, `_FAILED`, `_CREATED`, `_VALID`, `_CONSISTENT`, `_STARTED`, `_COMPLETED`

### 5.2 Nível semântico do LOG (`payload.level`)

O `level` interno do payload LOG é semântico — descreve o que o componente estava fazendo, não apenas a severidade:

```
LIFECYCLE_CHECK_STARTED
LIFECYCLE_CHECK_SUCCESS
LIFECYCLE_CHECK_FAILED
```

Isso é diferente do `level` do envelope (`INFO`, `ERROR`), que é a severidade para roteamento de infraestrutura.

---

## 6. Padrões de Eventos Observados no Domínio

Com base no log de produção, a DTS identifica três padrões recorrentes que devem ser formalizados:

### 6.1 Padrão Coordinator

Coordenadores emitem eventos de sucesso ou falha com `executionTimeMs` e `operation`:

```json
{
  "name": "SigningKeySourceCoordinator_SUCCESS",
  "data": {
    "executionTimeMs": 814,
    "status":          "COMPLETED",
    "operation":       "WARMUP_FETCH",
    "instanceId":      "f516c69ca51a",
    "traceId":         "6a3a893f847ed2d747f95ea7aaead2c2"
  }
}
```

**Campos obrigatórios neste padrão:** `executionTimeMs`, `status`, `operation`, `instanceId`, `traceId`.

### 6.2 Padrão Policy

Políticas emitem eventos de verificação com `conditionState` e `status`:

```json
{
  "name": "DUPLICATED_ACTIVE_KEY_POLICY_CHECK_CONSISTENT",
  "data": {
    "status":         "CONSISTENT",
    "operation":      "CHECK",
    "conditionState": "SATISFIED",
    "executionTimeMs": 0,
    "app":            "geo-sentinel-gateway",
    "instanceId":     "f516c69ca51a",
    "traceId":        "6a3a893f847ed2d747f95ea7aaead2c2"
  }
}
```

**Campos obrigatórios neste padrão:** `status`, `operation`, `conditionState`, `instanceId`, `traceId`.

### 6.3 Padrão Lifecycle

Eventos de ciclo de vida de entidades de domínio carregam o identificador da entidade e o motivo da transição:

```json
{
  "name": "SIGNING_KEY_CREATED",
  "data": {
    "keyId":          "rsa-1782221123620-3b803784",
    "rotationReason": "expiration",
    "instanceId":     "f516c69ca51a",
    "traceId":        "6a3a893f847ed2d747f95ea7aaead2c2"
  }
}
```

**Campos obrigatórios neste padrão:** identificador da entidade (`keyId`, `areaId`, `geofenceId`...), `instanceId`, `traceId`. O motivo da transição é recomendado quando aplicável.

---

## 7. Interoperabilidade com OpenTelemetry

A DTS não é OTel — mas é interoperável com OTel. A estratégia é de **exportação por consumidor**, não de adequação do payload.

### 7.1 O payload DTS permanece intacto no Event Store

O Event Store armazena o `Event()` no formato DTS canônico. Nunca no formato OTel. Isso preserva toda a semântica de domínio para consultas, reprocessamento e auditoria.

### 7.2 Consumidores exportadores traduzem para OTel

Um consumidor especializado — o **OTel Exporter** — lê eventos do Event Store e os converte para o formato OTLP, preservando a semântica em `attributes` com namespace próprio:

```json
{
  "timestamp":      "2026-06-23T13:25:25.808203761Z",
  "traceId":        "6a3a893f847ed2d747f95ea7aaead2c2",
  "severityNumber": 9,
  "severityText":   "INFO",
  "eventName":      "signing_key.created",
  "body":           "SIGNING_KEY_CREATED",
  "attributes": {
    "dts.source":              "SigningKeyLifecycleManagerImpl",
    "dts.key.id":              "rsa-1782221123620-3b803784",
    "dts.key.rotation_reason": "expiration",
    "dts.instance.id":         "f516c69ca51a"
  },
  "resource": {
    "service.name":            "geo-sentinel-gateway",
    "deployment.environment":  "production"
  }
}
```

O namespace `dts.*` nos `attributes` preserva a semântica GeoSentinel de forma queryável em qualquer backend OTel-compatível (Grafana, Datadog, Jaeger).

### 7.3 Mapeamento DTS → OTel

| Campo DTS | Campo OTel | Observação |
|---|---|---|
| `createdAt` | `Timestamp` | Converter ms → nanoseconds |
| `data.traceId` / `fields.traceId` | `TraceId` | Já em formato W3C hex |
| `data.spanId` / `fields.spanId` | `SpanId` | Se presente |
| `level` (envelope) | `SeverityNumber` / `SeverityText` | Mapeamento direto |
| `payload.name` | `eventName` (lowercase snake) | Ex: `SIGNING_KEY_CREATED` → `signing_key.created` |
| `payload.source` | `attributes["dts.source"]` | Preserva origem semântica |
| `data.*` / `fields.*` | `attributes["dts.*"]` | Todos os campos de domínio com namespace |
| `payload.message` | `Body` | Apenas para LOG |
| `id` | `attributes["dts.event.id"]` | Rastreabilidade do evento original |

---

## 8. Impacto no Observability SDK

### 8.1 API pública — sem mudança

```kotlin
observability.event(
  evidence   = SigningKeyCreatedEvidence(keyId = "...", rotationReason = "expiration"),
  persistence = PERSIST,
  dispatch    = DISPATCH
)
```

### 8.2 O SDK garante conformidade DTS automaticamente

O SDK é o guardião da especificação. Ele:

- Gera o `id` (UUID v4) automaticamente.
- Injeta `createdAt` no momento da chamada.
- Injeta `traceId` e `instanceId` do contexto de execução ativo.
- Valida que `name` segue `UPPER_SNAKE_CASE` sem valores dinâmicos.
- Serializa no formato DTS canônico antes de publicar no Kafka.

Nenhum produtor precisa conhecer a DTS internamente. O SDK garante conformidade por construção.

---

## 9. O que a DTS não especifica intencionalmente

**O conteúdo de `data` e `fields`** é livre. Cada domínio define seus próprios campos. A DTS não impõe um schema fixo para o payload de domínio — isso pertence ao Schema Registry do SDK, gerenciado por contrato entre produtor e consumidor.

**Políticas de persistência e dispatch** continuam nos headers Kafka — fora do payload, fora da DTS.

**Interpretação e agregação** continuam sendo responsabilidade dos consumidores.

---

## 10. Próximos Passos

1. **Validação retroativa:** verificar que todos os eventos do log de 23/06 estão em conformidade com a estrutura canônica da DTS. Os eventos observados já estão majoritariamente alinhados.
2. **SDK:** implementar validação de conformidade DTS em tempo de compilação — `name` sem valores dinâmicos, campos de correlação obrigatórios presentes.
3. **Schema Registry:** formalizar os três padrões (Coordinator, Policy, Lifecycle) como templates de `Evidence` no SDK.
4. **OTel Exporter:** implementar o consumidor exportador com o mapeamento `dts.*` descrito na seção 7.
5. **Documentação:** publicar o catálogo de eventos por serviço com seus padrões e campos obrigatórios.

---

*GeoSentinel Architecture RFC — Confidential*