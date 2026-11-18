# Documento de Desenho Técnico — Fase 5: Streaming

| | |
|---|---|
| **Projeto** | Olist Lakehouse |
| **Fase** | 5 — Streaming |
| **Responsável técnico** | João Guilherme Araujo |
| **Data** | 2026-11-16 |
| **Status** | Decisões S1 a S8 aprovadas em 2026-11-17 (ADRs 0009 a 0012); etapa 5.1 (simulador em modo `stream`) concluída em 2026-11-18 |
| **Documentos relacionados** | [Desenho da Fase 1 (simulador e bronze)](fase-1-simulador-bronze.md) · [Desenho da Fase 4 (gold)](fase-4-gold-metricas.md) · [ADR 0002](../adr/0002-pedido-valido-e-entrega-no-prazo.md) · [ADR 0007](../adr/0007-derivacao-de-status-e-definicoes-de-borda-da-gold.md) · [Dicionário de métricas](../dicionario_metricas.md) |

---

## 1. Objetivo deste documento

Registrar o desenho do monitor operacional **antes** da implementação: de onde vêm os eventos, como o estado do pedido é mantido sem ser corrompido por duplicatas e eventos fora de ordem, como se define "pedido em risco" e como o resultado é validado. As decisões S1 a S8 abaixo foram **aprovadas em 2026-11-17**, todas na opção recomendada, e registradas nos ADRs [0009](../adr/0009-simulador-em-modo-stream-e-relogio-fino.md) (S2, S8), [0010](../adr/0010-gold-orders-live-merge-idempotente-e-watermark.md) (S3, S4, S5), [0011](../adr/0011-regra-de-pedido-em-risco.md) (S6) e [0012](../adr/0012-sla-do-monitor-e-numero-oficial-versus-ao-vivo.md) (S1, S7). O texto de cada decisão fica como registro das opções avaliadas.

## 2. Contexto e alinhamento com o negócio

A visão do projeto é hoje retroativa: o `% de entregas no prazo` e o SLA de entrega só dizem que o atraso aconteceu, depois da entrega. A Logística quer agir **antes**: acionar o vendedor ou a transportadora enquanto o pedido ainda pode chegar a tempo.

| Área | Dor | O que a Fase 5 entrega |
|---|---|---|
| Logística | atrasos descobertos só depois da entrega | lista de pedidos em risco e já atrasados, atualizada a cada ciclo |
| Atendimento (CX) | cliente reclama antes de a empresa saber do atraso | mesma lista, para contato proativo |
| Diretoria | números mudam sem explicação quando chegam eventos tardios | regra explícita do que é "oficial" (batch) e do que é "ao vivo" |

**Critério de aceite do épico E5:** o evento chega na gold dentro do SLA definido, mesmo com eventos duplicados ou fora de ordem.

## 3. Estado de partida (dados em 2026-11-16)

| Fato | Valor |
|---|---|
| Simulação gerada até | **2028-08-21** (27 execuções, a partir de 2026-09-06); a referência de "hoje" é a data simulada, nunca `current_date()` |
| Dataset ainda não emitido | cerca de 2 meses (o dataset original termina em 2018-10, com o deslocamento de 3.654 dias) |
| `bronze.order_status_events` | 408.678 eventos de 98.474 pedidos; colunas `event_sequence`, `event_ts`, `status`, `order_id`, `sim_date` |
| Status atual dos pedidos (maior `event_sequence`) | `delivered` 93.247 · `shipped` 2.628 · `approved` 657 · `unavailable` 608 · `canceled` 584 · `invoiced` 314 · `processing` 301 · `created` 135 |
| Pedidos abertos (candidatos ao monitor) | 4.035: shipped, approved, invoiced, processing e created |
| Falhas injetadas pelo simulador | reenvio de arquivo 5%, registro duplicado 1%, atraso de chegada 3% de eventos e itens, de até 3 dias, e fora de ordem (consequência do atraso e do embaralhamento); tudo registrado no gabarito `ops.simulator_chaos_log` |
| Relógio do simulador | **diário** (uma pasta `sim_date=` por dia) |
| Gatilhos no serverless | `AvailableNow` suportado; `ProcessingTime` e `Continuous` não (erro `INFINITE_STREAMING_TRIGGER_NOT_SUPPORTED`) |

Consequência: o "tempo real" desta fase é uma execução **agendada e frequente** em modo `AvailableNow`, que processa tudo que chegou desde a última execução e termina.

## 4. Decisões a tomar

### S1 — SLA do monitor

**Pergunta:** o evento deve estar na gold em 5 ou em 15 minutos?

| Opção | Prós | Contras |
|---|---|---|
| 5 min | alerta mais rápido | muito mais execuções; cada uma paga a inicialização do serverless e consome cota diária do Free Edition |
| **15 min (recomendada)** | cabe na cota, deixa folga para a inicialização e a variação de duração | alerta menos fresco |

**Recomendação:** 15 minutos como SLA de projeto, medido entre o momento em que o arquivo chega à landing e o momento em que a linha aparece em `gold_orders_live`. Reavaliar na Fase 8 com a duração real de cada ciclo.

### S2 — Como o simulador emite os últimos 2 meses

**Pergunta:** o relógio é diário; o monitor precisa de granularidade menor. Como emitir o fim do dataset?

| Opção | Descrição |
|---|---|
| A | manter o dia como unidade e gerar **uma pasta por dia**, como hoje; o streaming processa dia a dia |
| **B (recomendada)** | novo modo `stream`: cada execução emite uma **fatia de horas simuladas** (por exemplo, 6 h), com um arquivo por fatia; o relógio fino passa a existir só para os 2 meses finais |
| C | gerar tudo de uma vez e fingir a chegada espaçada na leitura |

**Recomendação:** B. Preserva o simulador existente (modo `batch` intacto), cria eventos com instantes distintos dentro do dia e permite medir latência de verdade. Impacto: o nome da pasta passa a conter data e fatia, e o gabarito de falhas precisa registrar a fatia.

### S3 — Onde o streaming roda

**Pergunta:** notebook/job com Structured Streaming ou pipeline Lakeflow (streaming tables)?

**Recomendação:** **notebook/job com Structured Streaming e `foreachBatch`**, na linha da bronze (decisão B1 da Fase 1). O desafio da fase é justamente o `MERGE` idempotente dentro do `foreachBatch`, o que o pipeline declarativo esconde. A orquestração do job fica para a Fase 7.

### S4 — Modelo de `gold_orders_live`

Uma linha por `order_id`, mantida por `MERGE`, com: status atual, `event_sequence` e `event_ts` do último evento aplicado, o instante de cada etapa (aprovação, despacho, entrega), data estimada, `risco_atraso` (ver S6) e `_updated_at`. É uma **tabela Delta própria** (não materialized view), pois o estado é mutável e gravado por `MERGE`.

Regra de aplicação do `MERGE`: o evento só substitui o estado se tiver `event_sequence` **maior** que o já gravado; caso contrário, é ignorado. Isso torna a escrita idempotente e imune à ordem de chegada.

### S5 — Deduplicação e watermark

- `dropDuplicatesWithinWatermark` sobre `(order_id, event_sequence)`, com watermark sobre `event_ts`.
- **Tamanho do watermark:** o simulador atrasa até **3 dias**; o watermark precisa cobrir isso com margem (proposta: 4 dias). Evento mais atrasado que o watermark deixa de ser deduplicado em memória, mas **continua seguro**, pois o `MERGE` por sequência é a segunda linha de defesa.
- Política para evento tardio demais: aplicar pelo `MERGE` (que decide por `event_sequence`) e **contá-lo** em `ops` para auditoria; não descartar em silêncio.

### S6 — Regra de "pedido em risco"

**Pergunta:** o que define risco, em dias da **data simulada**?

Proposta, só para pedidos **abertos** (não `delivered`, `canceled` nem `unavailable`):

| Nível | Regra |
|---|---|
| `atrasado` | data estimada de entrega já passou na data simulada |
| `em_risco` | faltam até **N dias** para a data estimada e o pedido ainda não foi despachado (`shipped`), ou foi despachado e faltam menos dias do que o lead time mínimo observado |
| `normal` | demais |

**Pontos a decidir pelo João:** o valor de N (proposta: 2 dias) e se o lead time mínimo entra na regra ou fica para uma versão seguinte. A regra e os limites ficam em configuração, não no código.

### S7 — Relação com a gold batch e o dicionário

- O estado de `gold_orders_live` usa a **mesma derivação de status** da gold batch: maior `event_sequence` (ADR 0007).
- A gold batch (`fact_orders`) continua sendo a fonte **oficial** dos KPIs. `gold_orders_live` é a visão operacional ao vivo. Isso será dito no dicionário (nova versão) para evitar dois números diferentes sem explicação.

### S8 — Referência de "hoje"

O risco é calculado contra a **data simulada corrente** (a maior `sim_date`/`event_ts` já processada), nunca contra `current_date()`. Essa data é lida do estado do simulador.

## 5. Critérios de aceite (proposta)

1. `gold_orders_live` tem **uma linha por pedido** e nenhum `order_id` repetido.
2. **Equivalência com o batch:** para os pedidos até 2028-08-21, o status de `gold_orders_live` é igual ao de `fact_orders` em 100% dos casos.
3. **Duplicatas e fora de ordem:** com as falhas do gabarito (`ops.simulator_chaos_log`) aplicadas, o estado final é igual ao de uma execução sem falhas.
4. **Idempotência:** reprocessar os mesmos arquivos não muda `gold_orders_live` (mesma contagem e mesmo hash do estado).
5. **Latência:** o tempo entre a chegada do arquivo e a linha na gold fica dentro do SLA de S1 em todos os ciclos medidos.
6. **Risco:** a lista de `atrasado` e `em_risco` é igual a um cálculo independente em SQL sobre os mesmos dados.
7. **Evento tardio demais** é contado e visível, nunca descartado em silêncio.

## 6. Etapas previstas

| Etapa | Entrega |
|---|---|
| 5.1 | simulador em modo `stream` (S2) para os últimos 2 meses, com as falhas do gabarito |
| 5.2 | ingestão dos novos eventos na bronze e leitura em streaming |
| 5.3 | `gold_orders_live`: deduplicação, watermark e `MERGE` idempotente em `foreachBatch` (S4, S5) |
| 5.4 | regra de risco (S6), lista de pedidos em risco, validação independente (critérios 2 a 7) |
| 5.5 | dicionário v1.2 (S7), ADRs, documentação das tabelas e encerramento |

## 7. Riscos e pontos de atenção

- **Cota do Free Edition:** muitos ciclos curtos consomem a cota diária. A execução dos 2 meses finais deve ser feita em lotes de fatias, não em um agendamento real de 15 em 15 minutos.
- **Mudança retroativa de KPI:** com os 2 meses finais, o GMV e a taxa de cancelamento de meses antigos podem mudar (cancelamento tardio). A validação da Fase 4 precisará distinguir "dados até 2028-08-21" de "dados completos".
- **Evento de pedido que o cabeçalho ainda não conhece:** por desenho do simulador, o cabeçalho (`orders`) nunca atrasa; itens e eventos podem. O risco inverso (evento órfão) não deve aparecer, e deve ser verificado.
- **Dados pessoais:** `gold_orders_live` não precisa de nenhuma coluna de cadastro; manter assim.

## 8. Resumo das decisões a aprovar

| # | Decisão | Recomendação |
|---|---|---|
| S1 | SLA do monitor | 15 minutos |
| S2 | Emissão dos 2 meses finais | modo `stream`, fatias de horas simuladas |
| S3 | Onde roda | notebook/job com `foreachBatch` |
| S4 | Modelo de `gold_orders_live` | uma linha por pedido, `MERGE` por `event_sequence` maior |
| S5 | Dedup e watermark | `dropDuplicatesWithinWatermark`, watermark de 4 dias, `MERGE` como segunda defesa |
| S6 | Pedido em risco | abertos; `atrasado` e `em_risco` (N = 2 dias), regra em configuração |
| S7 | Batch × ao vivo | batch é o oficial; ao vivo é operacional; dicionário v1.2 |
| S8 | "Hoje" | data simulada corrente |
