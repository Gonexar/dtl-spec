# DTL-VIS: Domain Telemetry Language Visual Specification
## Especificação Canônica de Visualização para Domain Telemetry Language

**Status:** PROPOSED  
**Autor:** Engenharia de Arquitetura GeoSentinel  
**Data:** 24 de Junho de 2026  
**Versão:** v1.0  
**Relacionado:** Domain Telemetry Language (DTL) RFC v1.0  
**Implementação de Referência:** Heat Map gerado por LLM a partir do log de produção de 23/06/2026

---

## 1. Propósito

A DTL-VIS define como eventos Domain Telemetry Language se traduzem em representação visual de forma **determinística**. Dado o mesmo conjunto de eventos DTL, qualquer implementação conforme a esta especificação deve produzir o mesmo diagrama — mesma geometria, mesmas cores, mesmos símbolos, mesma semântica visual.

A especificação serve três audiências:

**Engenheiros e operadores humanos** — um vocabulário visual aprendido uma vez, reconhecível em qualquer sistema que adote DTL.

**Agentes LLM** — uma gramática visual formal que permite a um modelo gerar, interpretar e raciocinar sobre diagramas DTL sem ambiguidade.

**Implementadores de ferramentas** — um contrato preciso para construir renderers, exportadores e dashboards compatíveis com DTL-VIS.

---

## 2. Anatomia do Diagrama

O diagrama DTL-VIS é composto por seis elementos estruturais fixos:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  TÍTULO DO DIAGRAMA                              LEGENDA DE DURAÇÃO (Heat Scale) │
│  subtítulo descritivo                            LEGENDA DE SÍMBOLOS             │
├──────────┬──────────────┬──────────┬─────────────────┬──────────┬───────────────┤
│  TEMPO   │ ORQUESTRAÇÃO │  STATE   │   EXECUÇÕES     │ CACHE L1 │ CACHE L2 │ F/D│
│          │  / DOMÍNIO   │ (AVALIA- │  (COMPLETION)   │  (APP)   │ (SHARED) │    │
│          │              │  ÇÕES)   │                 │          │          │    │
├──────────┼──────────────┼──────────┼─────────────────┼──────────┼──────────┼────┤
│          │              │          │                 │          │          │    │
│  linhas  │    eventos   │ eventos  │    eventos      │ eventos  │ eventos  │ DE │
│  de      │    de        │ de       │    de           │ de       │ de       │    │
│  tempo   │    LOG +     │ POLICY   │    COORDINATOR  │ CACHE    │ CACHE    │    │
│          │    Lifecycle │          │                 │ APP      │ SHARED   │    │
│          │              │          │                 │          │          │    │
├──────────┴──────────────┴──────────┴─────────────────┴──────────┴──────────┴────┤
│  PAINEL DE RODAPÉ: Legenda | Estatísticas | Performance | Insights               │
└─────────────────────────────────────────────────────────────────────────────────┘
```
<img width="1536" height="1024" alt="ChatGPT Image 22 de jun  de 2026, 23_06_04" src="https://github.com/user-attachments/assets/61d909f7-af44-4d5d-9974-bcb4cf2182bc" />
<img width="1536" height="1024" alt="ChatGPT Image 27 de jun  de 2026, 07_58_44" src="https://github.com/user-attachments/assets/1e6e087d-7a1f-44dd-b86a-cc2573b89979" />

---

## 3. Eixos

### 3.1 Eixo Y — Tempo (obrigatório)

O eixo Y representa tempo absoluto. É sempre a primeira coluna do diagrama.

**Formato de cada célula de tempo:**

```
HH:MM:SS.mmm     ← timestamp absoluto (createdAt do primeiro evento do grupo)
+Xms / +Xs       ← offset relativo ao início do trace (ou do diagrama)
```

**Regras:**
- O timestamp absoluto deriva de `createdAt` convertido para formato humano legível.
- O offset relativo é calculado a partir do `createdAt` do primeiro evento do diagrama.
- Linhas de tempo são criadas por **grupo de eventos simultâneos ou próximos** — eventos dentro de um mesmo instante lógico compartilham a mesma linha.
- A granularidade mínima é 1ms. A máxima exibida é segundos inteiros para offsets acima de 10s.

**Separação de fluxos:**
Quando múltiplos `traceId` distintos estão presentes, o eixo Y inclui **separadores visuais horizontais** com rótulo do nome do fluxo. O rótulo é derivado do evento LOG `_STARTED` do trace.

```
← ROTAÇÃO DE CHAVES ────────────────────────────────────────────────────────── →
← USER CONTEXT FLOW ─────────────────────────────────────────────────────────── →
```

### 3.2 Eixo X — Camadas do Sistema (obrigatório)

O eixo X representa as camadas arquiteturais do sistema. A ordem é **fixa e canônica** — da esquerda para a direita, do mais abstrato para o mais concreto:

| Posição | Coluna | Conteúdo | Ícone |
|---|---|---|---|
| 1 | ORQUESTRAÇÃO / DOMÍNIO | Eventos LOG (narrativos) e Lifecycle Events | ⭐ |
| 2 | STATE (AVALIAÇÕES) | Eventos de POLICY pattern | 🛡 |
| 3 | EXECUÇÕES (COMPLETION) | Eventos de COORDINATOR pattern | ⚙ |
| 4 | CACHE L1 (APP) | Eventos de cache de aplicação | 🗄 |
| 5 | CACHE L2 (SHARED) | Eventos de cache compartilhado | 🗄 |
| 6 | FONTE / DESTINO | Chamadas externas, APIs, Domain Events resultantes | 💛 |

**Regra de posicionamento:**
O `source` do evento DTL determina a coluna. O mapeamento `source → coluna` é inferido pelo padrão do nome:

- Contém `Scheduler`, `Manager`, `Lifecycle` → ORQUESTRAÇÃO
- Contém `Policy`, `Validator`, `Evaluator` → STATE
- Contém `Coordinator` (não Cache) → EXECUÇÕES
- Contém `AppCache` ou `L1` → CACHE L1
- Contém `SharedCache` ou `L2` → CACHE L2
- `SIGNING_AUTHORITY`, `USER_CONTEXT_SERVICE`, Domain Events → FONTE/DESTINO

---

## 4. Símbolos e Nós

### 4.1 Três tipos de nó — derivados do pattern DTL

Cada nó no diagrama corresponde a um evento DTL. O tipo visual do nó é determinado pelo pattern do componente:

#### COMPLETION (Execução)
**Cor da borda:** Azul (`#4FC3F7`)  
**Forma:** Retângulo com bordas arredondadas  
**Conteúdo:**
```
┌─────────────────────────────┐
│  NomeDoComponente           │  ← payload.source
│  NOME_DA_OPERACAO           │  ← data.operation
│  duração destacada          │  ← data.executionTimeMs (cor = heat scale)
└─────────────────────────────┘
```
**Quando usar:** Eventos do pattern Coordinator — qualquer `source` que termine em `Coordinator` ou `Container`.

#### STATE (Avaliação)
**Cor da borda:** Verde (`#66BB6A`)  
**Forma:** Retângulo com bordas arredondadas  
**Conteúdo:**
```
┌─────────────────────────────┐
│  NOME_DA_POLICY             │  ← payload.source
│  OPERACAO                   │  ← data.operation
│  STATUS                     │  ← data.status ou conditionState
│  duração                    │  ← data.executionTimeMs
└─────────────────────────────┘
```
**Quando usar:** Eventos do pattern Policy — qualquer `source` que contenha `POLICY`, `VALIDATOR`, ou `EVALUATOR`.

**Nota especial:** Quando `conditionState: UNSATISFIED`, a borda muda para laranja (`#FFA726`) — não é erro, mas é um estado que dispara ação. O texto do status é renderizado em laranja.

#### DOMAIN EVENT
**Cor da borda:** Roxo/Lilás (`#CE93D8`)  
**Forma:** Retângulo com bordas arredondadas, fundo levemente mais escuro  
**Conteúdo:**
```
┌─────────────────────────────┐
│  NOME_DO_EVENTO             │  ← payload.name
│  DOMAIN EVENT               │  ← rótulo fixo
└─────────────────────────────┘
```
**Quando usar:** Eventos do pattern Lifecycle — `SIGNING_KEY_CREATED`, `SIGNING_KEY_RETIRING`, `USER_CONTEXT_RETURNED`, `INSTANCE_REGISTERED`, etc.

**Posicionamento:** Sempre na coluna FONTE / DESTINO, alinhado à direita do diagrama.

### 4.2 Nó especial — LOG Narrativo

Eventos do tipo `LOG` (payloadType: LOG) são renderizados na coluna ORQUESTRAÇÃO com estilo distinto:

```
┌─────────────────────────────┐
│  NomeDoScheduler            │  ← payload.source
│  NIVEL_SEMANTICO            │  ← payload.level (ex: LIFECYCLE_CHECK_STARTED)
│  0 ms                       │  ← sempre zero ou omitido
└─────────────────────────────┘
```

LOGs `_STARTED` e `_SUCCESS` funcionam como **marcadores de abertura e fechamento** da narrativa visual. São conectados por uma linha vertical tracejada ao longo de toda a altura do fluxo.

### 4.3 Nó especial — CACHE MISS / HIT

Quando um evento de cache retorna miss (inferido por ausência de resultado seguida de busca externa):

```
┌─────────────────────────────┐
│  CACHE MISS                 │  ← rótulo semântico
│  (Context not found)        │  ← mensagem descritiva
└─────────────────────────────┘
```

Borda vermelha suave (`#EF5350`). Posicionado na coluna correspondente ao tipo de cache.

---

## 5. A Escala de Calor (Heat Scale)

A escala de calor é o elemento mais crítico da DTL-VIS. É o que transforma um diagrama de fluxo em um mapa de performance.

### 5.1 Escala canônica de duração

| Faixa | Cor | Hex | Semântica |
|---|---|---|---|
| 0ms | Verde escuro | `#1B5E20` | Instantâneo — cache hit, operação em memória |
| 1ms – 10ms | Verde | `#4CAF50` | Muito rápido — esperado para validações e políticas |
| 10ms – 50ms | Verde amarelado | `#8BC34A` | Rápido — operações locais normais |
| 50ms – 200ms | Amarelo | `#FFEB3B` | Aceitável — operações de rede locais |
| 200ms – 1s | Laranja | `#FF9800` | Lento — investigar em análise de performance |
| 1s – 2s | Laranja escuro | `#F4511E` | Muito lento — candidato a otimização |
| 2s+ | Vermelho | `#B71C1C` | Crítico — degradação de performance |

**A cor de duração é aplicada ao texto do valor de `executionTimeMs`** dentro do nó, não ao fundo do nó. O fundo permanece escuro e neutro.

### 5.2 Renderização da duração

```
// Abaixo de 1 segundo
204 ms      ← cor amarela
62 ms       ← cor verde amarelado
0 ms        ← cor verde escuro

// Acima de 1 segundo — sempre com unidade explícita
1.185 s     ← cor laranja escuro
2.127 s     ← cor vermelho
```

A barra de legenda da escala de calor aparece **obrigatoriamente** no canto superior direito do diagrama, com os marcos: `0ms`, `10ms`, `50ms`, `200ms`, `1s`, `2s+`.

---

## 6. Conexões e Fluxo

### 6.1 Setas horizontais — sequência de execução

Eventos dentro do mesmo `traceId` que ocorrem em sequência dentro da mesma linha de tempo são conectados por **setas horizontais** da esquerda para a direita.

```
[STATE] ──→ [COORDINATOR] ──→ [CACHE L1] ──→ [CACHE L2] ──→ [DOMAIN EVENT]
```

**Regra de conexão:** dois eventos são conectados horizontalmente quando:
- Compartilham o mesmo `traceId`
- O segundo evento `createdAt` é imediatamente posterior ao primeiro
- Estão em colunas adjacentes ou na mesma linha de tempo

### 6.2 Setas verticais — progressão temporal

Quando o fluxo avança para a próxima linha de tempo dentro do mesmo trace, uma **seta vertical ou diagonal** conecta o último nó da linha anterior ao primeiro nó da linha seguinte.

### 6.3 Linha tracejada — fronteira de fluxo

Quando `conditionState: UNSATISFIED` dispara uma ação subsequente, a conexão entre o nó STATE e o próximo nó de execução usa uma **seta tracejada** em laranja — sinalizando causalidade condicional, não sequência direta.

```
[STATE UNSATISFIED] - - → [COORDINATOR execução disparada]
```

### 6.4 Separador de fluxo — múltiplos traces

Quando o diagrama contém múltiplos `traceId`, uma **linha horizontal divisória** com rótulo de texto separa os fluxos:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
← ROTAÇÃO DE CHAVES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
← USER CONTEXT FLOW
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

O rótulo do fluxo é derivado do `message` do LOG `_STARTED` do trace, ou do `name` do primeiro evento Lifecycle.

---

## 7. Painel de Rodapé

O rodapé é **obrigatório** e composto por quatro blocos fixos:

### 7.1 Legenda de Símbolos

```
● COMPLETION: Execuções que produzem resultados
● STATE: Avaliações de condições/estados  
● DOMAIN EVENT: Eventos de domínio gerados
```

### 7.2 Estatísticas do Fluxo

Derivadas dos eventos DTL por `traceId`:

```
[Nome do Trace 1]:
  Total de eventos: N
  Tempo total: Xms / Xs

[Nome do Trace 2]:
  ...
```

### 7.3 Análise de Performance (Tempo Total)

Barra horizontal proporcional mostrando a distribuição de `executionTimeMs` por categoria de componente:

| Categoria | Derivada de |
|---|---|
| Criptografia | `source` contém `Vault` |
| Rede/Fonte | `source` contém `Source`, `Provider`, `Response` |
| Cache Ops | `source` contém `Cache` |
| Regras/Validações | `source` contém `Policy`, `Validator`, `Evaluator` |

Cada categoria é renderizada como segmento colorido com percentual e tempo absoluto.

### 7.4 Insights

Lista de observações geradas automaticamente a partir dos dados:

**Regras de geração de insights:**

- Se algum componente representa mais de 40% do tempo total → *"X é o principal consumidor de tempo"*
- Se cache miss seguido de cache hit reduz tempo em mais de 80% → *"Cache hit reduz latência em X% (Y→Zms)"*
- Se `conditionState: UNSATISFIED` disparou execução → *"Condição Y disparou rotação/execução"*
- Se todos os STATE events retornam em 0-1ms → *"Avaliações de política sem impacto de performance"*
- Se dois fluxos distintos operam corretamente → *"Fluxos X e Y operando corretamente"*

---

## 8. Tema Visual

### 8.1 Paleta de cores do diagrama

| Elemento | Cor | Hex |
|---|---|---|
| Fundo do diagrama | Preto azulado escuro | `#0D1117` |
| Fundo dos nós | Cinza escuro | `#161B22` |
| Texto primário | Branco | `#E6EDF3` |
| Texto secundário | Cinza claro | `#8B949E` |
| Separadores e bordas | Cinza médio | `#30363D` |
| Borda COMPLETION | Azul | `#4FC3F7` |
| Borda STATE (SATISFIED) | Verde | `#66BB6A` |
| Borda STATE (UNSATISFIED) | Laranja | `#FFA726` |
| Borda DOMAIN EVENT | Lilás | `#CE93D8` |
| Borda CACHE MISS | Vermelho suave | `#EF5350` |
| Separador de fluxo | Branco com alpha | `#FFFFFF40` |
| Setas normais | Cinza claro | `#8B949E` |
| Setas condicionais | Laranja | `#FFA726` |
| Rótulo de fluxo | Roxo claro | `#C084FC` |

### 8.2 Tipografia

| Elemento | Tamanho | Peso |
|---|---|---|
| Título do diagrama | 18px | Bold |
| Subtítulo | 12px | Regular |
| Nome do componente (nó) | 11px | Medium |
| Operação (nó) | 11px | Bold |
| Duração (nó) | 12px | Bold (cor da heat scale) |
| Timestamp (eixo Y) | 11px | Mono Regular |
| Offset (eixo Y) | 10px | Mono Regular, cor amarela |
| Cabeçalho de coluna | 11px | Bold uppercase |
| Rótulo de fluxo | 11px | Bold, cor roxa |

---

## 9. Algoritmo de Renderização

Um renderer DTL-VIS conforme deve seguir esta sequência:

```
1. PARSE
   └── Ler eventos DTL
   └── Agrupar por traceId
   └── Ordenar por createdAt dentro de cada grupo

2. CLASSIFY
   └── Para cada evento, determinar:
       ├── Tipo de nó (COMPLETION | STATE | DOMAIN EVENT | LOG)
       ├── Coluna de destino (por source pattern)
       └── Cor de duração (por executionTimeMs e heat scale)

3. GROUP
   └── Agrupar eventos em linhas de tempo
       └── Eventos dentro de 5ms do mesmo trace → mesma linha
       └── Eventos acima de 5ms de diferença → nova linha

4. LAYOUT
   └── Calcular posições X/Y de cada nó
   └── Garantir que colunas respeitem a ordem canônica
   └── Adicionar separadores entre traces

5. CONNECT
   └── Traçar setas horizontais entre eventos da mesma linha
   └── Traçar setas verticais entre linhas do mesmo trace
   └── Marcar setas condicionais onde conditionState: UNSATISFIED precede execução

6. ANNOTATE
   └── Aplicar heat scale ao texto de duração
   └── Marcar CACHE MISS quando inferível
   └── Adicionar rótulos de fluxo nos separadores

7. RENDER FOOTER
   └── Calcular estatísticas por traceId
   └── Calcular distribuição de performance por categoria
   └── Gerar insights automáticos pelas regras da seção 7.4

8. COMPOSE
   └── Montar título, legenda de heat scale, legenda de símbolos
   └── Renderizar diagrama completo
```

---

## 10. Regras de Conformidade

Uma implementação é **DTL-VIS conforme** quando:

| # | Regra |
|---|---|
| R1 | A ordem das colunas é sempre: Orquestração → State → Execuções → Cache L1 → Cache L2 → Fonte/Destino |
| R2 | A escala de calor usa exatamente as sete faixas e cores da seção 5.1 |
| R3 | Eventos do mesmo `traceId` são sempre agrupados no mesmo fluxo |
| R4 | `conditionState: UNSATISFIED` gera seta condicional tracejada em laranja |
| R5 | Domain Events (Lifecycle) aparecem sempre na coluna Fonte/Destino |
| R6 | Todo diagrama inclui o painel de rodapé com os quatro blocos obrigatórios |
| R7 | O texto de duração usa a cor da heat scale, não o fundo do nó |
| R8 | LOGs `_STARTED` e `_SUCCESS` do mesmo trace são conectados por linha tracejada vertical |
| R9 | Múltiplos traces são separados por linha horizontal com rótulo derivado dos eventos |
| R10 | A legenda de heat scale aparece no canto superior direito com os seis marcos de referência |

---

## 11. Prompt de Referência para Agentes LLM

Para que um agente LLM gere um diagrama DTL-VIS conforme, o seguinte prompt de sistema deve ser fornecido:

```
Você está gerando um diagrama DTL-VIS (Domain Telemetry Language Visual Specification).

Regras obrigatórias:
- Eixo Y: tempo (createdAt convertido), com offset relativo ao primeiro evento
- Eixo X: colunas fixas nesta ordem: Orquestração/Domínio | State (Avaliações) | 
  Execuções (Completion) | Cache L1 (App) | Cache L2 (Shared) | Fonte/Destino
- Três tipos de nó: COMPLETION (borda azul), STATE (borda verde/laranja), 
  DOMAIN EVENT (borda lilás)
- Heat scale de duração: verde (0-10ms) → amarelo (50-200ms) → 
  laranja (1s) → vermelho (2s+)
- conditionState: UNSATISFIED → seta tracejada laranja para próxima execução
- Eventos do mesmo traceId formam um fluxo; múltiplos traces são separados 
  por linha horizontal com rótulo
- Rodapé obrigatório: Legenda | Estatísticas | Performance | Insights
- Tema: fundo escuro (#0D1117), texto claro, bordas de nó coloridas por tipo
- Insights gerados automaticamente: gargalos (>40% tempo), cache impact, 
  condições disparadas, estados de política
```

---

*GeoSentinel Architecture — DTL-VIS Specification v1.0 — Confidential*
