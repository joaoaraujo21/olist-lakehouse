# ADR 0015 — Leitura em streaming da bronze com `skipChangeCommits`

- **Status:** aceito
- **Data:** 2026-11-18
- **Fase:** 5 — Streaming

## Contexto

A bronze é Delta *append-only*, e `gold_orders_live` a lê como fonte de streaming. Uma fonte Delta de streaming aceita só acréscimos: qualquer `UPDATE`, `DELETE` ou `MERGE` na tabela de origem derruba a leitura com `DELTA_SOURCE_TABLE_IGNORE_CHANGES`. A etapa 5.2 reproduziu o erro numa cópia (`SHALLOW CLONE`) da bronze, depois de um `DELETE`.

A Fase 6 (governança e LGPD) prevê a **exclusão de cliente ponta a ponta**, que apaga linhas da bronze. Sem tratamento, a primeira exclusão pararia o monitor.

## Decisão

1. A leitura de produção usa `skipChangeCommits = true`: o stream **ignora** os commits que reescrevem linhas já existentes e lê só o que foi acrescentado. Testado na etapa 5.2: o `DELETE` na cópia foi ignorado e só as 3 linhas inseridas chegaram ao stream.
2. A exclusão de cliente **não chega sozinha** a `gold_orders_live`. Isso é aceito porque a tabela não guarda dado pessoal (só `order_id`, status, instantes e data estimada, sem cliente, vendedor, cidade nem CEP); a Fase 6 trata os dados pessoais nas camadas que os têm (bronze, silver e gold com cadastro).
3. Se uma exclusão precisar refletir no estado ao vivo (por exemplo, apagar um pedido inteiro), a rota é reconstruir `gold_orders_live` relendo a bronze com um checkpoint novo: o `MERGE` idempotente torna isso seguro, e a conferência da etapa 5.4 prova que o replay dá o mesmo estado.

## Alternativas consideradas

1. **Não usar a opção.** Mais simples, mas qualquer exclusão derruba o monitor até alguém recriar o checkpoint.
2. **Change Data Feed da bronze como fonte.** Entregaria as exclusões ao stream, mas exige tratar `delete` no `MERGE` e habilitar o feed na tabela da Fase 1; é mais peso do que o monitor precisa, já que o estado não tem dado pessoal.
3. **Reconstruir o estado a cada ciclo a partir da bronze.** Resolve tudo, porém custa a carga inteira (cerca de 70 s e 390 mil eventos) em vez de poucas dezenas de eventos por ciclo.

## Consequências

- (+) O monitor não para quando a bronze sofre `DELETE` ou `UPDATE`.
- (+) Comportamento coberto por teste repetível (`teste_leitura = robustez`).
- (−) Pedidos apagados da bronze continuam em `gold_orders_live` até a reconstrução; a Fase 6 deve incluir essa reconstrução, ou a exclusão do pedido no estado, no procedimento de exclusão de cliente.
- (−) O caso sem a opção (erro) foi observado e documentado, mas não é afirmado pelo teste: o Databricks encerra o comando quando qualquer stream termina com erro, mesmo tratado.
