# ADR 0010 — `gold_orders_live`: MERGE idempotente, deduplicação e watermark

- **Status:** aceito
- **Data:** 2026-11-17
- **Fase:** 5 — Streaming

## Contexto

O desafio central da fase: eventos duplicados e fora de ordem não podem corromper o estado do pedido. O simulador atrasa eventos em até 3 dias, reenvia arquivos (5%) e duplica registros (1%). O serverless do Free Edition só suporta o gatilho `AvailableNow`.

## Decisão

1. **Onde roda:** notebook/job com Structured Streaming e `foreachBatch`, na linha da bronze (decisão B1 da Fase 1). Gatilho `AvailableNow`, agendado com frequência. A orquestração em job fica para a Fase 7.
2. **Modelo:** `gold_orders_live` é uma **tabela Delta própria**, uma linha por `order_id`, mantida por `MERGE`. Colunas: status atual, `event_sequence` e `event_ts` do último evento aplicado, instante de cada etapa (aprovação, despacho, entrega), data estimada, `risco_atraso` (ADR 0011) e `_updated_at`. Sem colunas de cadastro.
3. **Regra do `MERGE`:** o evento só substitui o estado se tiver `event_sequence` **maior** que a já gravada; caso contrário é ignorado. A escrita é idempotente e não depende da ordem de chegada.
4. **Deduplicação:** `dropDuplicatesWithinWatermark` sobre `(order_id, event_sequence)`, com watermark de **4 dias** sobre `event_ts` (o atraso injetado é de 3 dias, mais 1 de margem).
5. **Evento mais atrasado que o watermark** deixa de ser deduplicado em memória, mas continua seguro, pois o `MERGE` por sequência é a segunda defesa. Ele é aplicado e **contado em `ops`**, nunca descartado em silêncio.
6. **Mesma derivação de status da gold batch:** maior `event_sequence` (ADR 0007).

## Alternativas consideradas

1. **Pipeline Lakeflow com streaming tables.** Esconde justamente o `MERGE` idempotente, que é o foco de aprendizado da fase, e dá menos controle da regra por sequência.
2. **Escrever o stream direto em tabela (append) e derivar o estado por view.** Simples, mas reprocessa todos os eventos a cada leitura e não entrega uma tabela de estado barata de consultar.
3. **Materialized view sobre a bronze.** É o que a gold batch já faz; não serve para estado ao vivo, pois depende de atualização do pipeline e não mostra o `MERGE` nem o watermark.
4. **Só deduplicar com watermark, sem a regra por sequência.** Falha para eventos tardios além do watermark.
5. **Watermark maior (7 dias ou mais).** Mais estado em memória sem ganho, dado o atraso máximo medido.

## Consequências

- (+) O estado final do pedido é o mesmo com ou sem duplicatas e desordem; reprocessar os mesmos arquivos não muda a tabela.
- (+) Duas linhas de defesa: o watermark reduz custo, o `MERGE` garante a correção.
- (−) A tabela é mutável e fica fora do pipeline declarativo, então a governança (Fase 6) e o job (Fase 7) precisam tratá-la explicitamente.
- (−) Se o atraso do simulador aumentar, o watermark precisa ser revisto; ele deve ficar em configuração.
