# ADR 0013 — Relógio do watermark: instante de chegada à bronze

- **Status:** aceito
- **Data:** 2026-11-18
- **Fase:** 5 — Streaming
- **Relação:** substitui os itens 4 e 5 da decisão do [ADR 0010](0010-gold-orders-live-merge-idempotente-e-watermark.md); os demais itens do ADR 0010 continuam valendo.

## Contexto

O ADR 0010 decidiu deduplicar com `dropDuplicatesWithinWatermark` sobre `(order_id, event_sequence)`, com watermark de 4 dias contado sobre o `event_ts` (tempo do evento), e afirmou que um evento mais velho que o watermark "continua seguro", pois o `MERGE` por sequência o aplicaria e ele seria contado em `ops`.

A medição da etapa 5.3, em tabelas de prova, mostrou que isso **não acontece**: com o `event_ts` como relógio, o Spark **descarta** todo evento mais velho que o watermark, antes do `MERGE`. Um evento novo (por exemplo, um `approved` atrasado) nunca chega ao estado e nenhuma contagem o registra. Dos três eventos tardios do teste (um distinto e dois duplicados), nenhum chegou ao `MERGE`; o pedido ficou com o status anterior. Isso quebra o critério 7 do desenho ("evento tardio demais é contado e visível, nunca descartado em silêncio").

Nos dados reais do simulador o problema não aparece porque o atraso máximo (3 dias) é menor que o watermark (4 dias), mas a regra do ADR 0010 só valeria por acaso dessa margem.

## Decisão

1. O relógio do watermark é o **instante de chegada do evento à bronze** (`_ingested_at`), não o `event_ts`. O valor só avança, então nenhum evento chega "atrasado demais" em relação a ele; nenhum é descartado pelo Spark.
2. O watermark continua de **4 dias**, agora como tempo que o estado de deduplicação lembra de cada chave já vista. Duplicado que chega depois disso passa pela deduplicação e é ignorado pelo `MERGE` por `event_sequence` (a segunda defesa continua valendo e agora é de fato alcançada).
3. **Evento tardio** é o evento com `event_ts` mais antigo que (maior `event_ts` já gravado em `gold_orders_live` menos 4 dias). Ele é **aplicado** pelo `MERGE` e **contado** em `ops.streaming_batches.late_events`.
4. A carga inicial lê toda a bronze como um único micro-lote do próprio stream; o `MERGE` idempotente torna seguro recomeçar com checkpoint novo.

## Alternativas consideradas

1. **Manter o `event_ts` e contar os tardios antes da deduplicação, num segundo stream.** Mede o descarte, mas não o evita: o evento continuaria perdido, e o custo de leitura dobraria.
2. **Tirar a deduplicação do stream e deixar só o `MERGE`.** Correto, porém reprocessa todo duplicado e abandona a deduplicação com estado, que é parte do que a fase quer demonstrar.
3. **Aumentar o watermark (7 dias ou mais) sobre o `event_ts`.** Adia o problema sem resolvê-lo: qualquer evento mais velho que o novo limite continua sendo descartado.
4. **Usar `_source_file_modified_at` (instante de gravação do arquivo) como relógio.** Equivalente na prática, mas reenvios de arquivo têm data de gravação mais nova que o original, o que não ajuda; `_ingested_at` é a medida da própria bronze.

## Consequências

- (+) Nada é descartado pelo Spark: evento atrasado é aplicado e contado, e o estado final do pedido não depende da ordem nem do atraso.
- (+) A medição virou teste repetível (`teste_leitura = watermark`), que mostra os dois relógios lado a lado.
- (−) O relógio é o de chegada, não o de negócio: serviria mal para agregações por janela de tempo do evento. A fase não tem esse caso.
- (−) No projeto, o tempo real dos ciclos (horas ou dias) não acompanha o tempo simulado; a janela de 4 dias de chegada cobre toda a execução e o estado de deduplicação não esquece nenhuma chave (cerca de 392 mil, pequeno). Num sistema real, em que a chegada acompanha o relógio, a janela de 4 dias passa a limitar o estado de fato.
