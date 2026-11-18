# Fase 5 — Streaming

- **Período:** iniciada em 2026-11-16
- **Status:** ✅ concluída em 2026-11-18 — [desenho técnico](../design/fase-5-streaming.md) (S1 a S8) implementado nas etapas 5.1 a 5.5
- **Branch/PRs:** `fase-5` (5.1), `fase-5.2/bronze-streaming` (5.2), `fase-5.3/gold-orders-live` (5.3), `fase-5.4/risco-e-validacao` (5.4) e `fase-5.5/encerramento` (5.5); um PR por etapa

## Objetivo

**Negócio:** a logística quer saber dos **pedidos em risco antes de atrasarem**, em vez de descobrir o atraso depois da entrega (hoje a visão é só retroativa). Aceite do épico E5: o evento chega na gold dentro do SLA definido, mesmo com eventos duplicados ou fora de ordem.

**Técnico:** Structured Streaming sobre o feed de eventos de status, com watermark, deduplicação e `foreachBatch` + `MERGE` idempotente, mantendo o estado atual de cada pedido em `gold_orders_live` e uma lista de pedidos em risco de atraso.

**Desafio central:** eventos duplicados e fora de ordem não podem corromper o estado do pedido.

O contexto, o escopo e os critérios de aceite estão no [desenho técnico](../design/fase-5-streaming.md).

## Ponto de partida

- A simulação foi gerada até **2028-08-21** (27 execuções, de 2026-09-06 a 2028-08-21). Os **últimos cerca de 2 meses do dataset** ainda não foram emitidos e são o material da Fase 5.
- Na data simulada atual há **4.035 pedidos abertos** (shipped 2.628, approved 657, invoiced 314, processing 301, created 135): é a base para o monitor de risco.
- `bronze.order_status_events` tem 408.678 eventos de 98.474 pedidos, com `event_sequence` e `event_ts`; o status atual dos pedidos já é derivado pela gold batch pela maior `event_sequence` (ADR 0007).
- Restrição do ambiente: o serverless suporta o gatilho `AvailableNow`, mas não `ProcessingTime` nem `Continuous` (ver [desenho da Fase 1](../design/fase-1-simulador-bronze.md)). O "quase tempo real" será uma execução agendada frequente.

## Pendências herdadas da Fase 4

Ver "Encerramento" em [fase-4-gold-metricas.md](fase-4-gold-metricas.md):

- `gold_orders_live` e o monitor de pedidos em risco (esta fase);
- os últimos 2 meses do dataset (esta fase);
- taxa de cancelamento e atraso passam a mudar conforme chegam eventos: o GMV de um mês pode diminuir quando um pedido válido é cancelado. O dicionário precisa dizer qual número é "oficial" (o batch) e qual é "ao vivo";
- SLA do monitor (5 vs 15 min), decisão aberta do projeto (S1 do desenho).

Ficam para as fases seguintes: governança e LGPD (Fase 6), testes, CI/CD e orquestração (Fase 7), performance (Fase 8).

## Etapas (previstas)

| Etapa | Entrega | Status |
|---|---|---|
| 5.1 | Simulador em modo "stream": últimos 2 meses, relógio fino, com duplicatas, atraso e fora de ordem | ✅ concluída (2026-11-18) |
| 5.2 | Ingestão dos novos eventos na bronze e leitura em streaming | ✅ concluída (2026-11-18) |
| 5.3 | `gold_orders_live`: deduplicação com watermark e `MERGE` idempotente em `foreachBatch` | ✅ concluída (2026-11-18) |
| 5.4 | Regra de risco de atraso, lista de pedidos em risco e validação independente | ✅ concluída (2026-11-18) |
| 5.5 | Documentação, ADRs e encerramento | ✅ concluída (2026-11-18) |

## O que foi feito

- 2026-11-16 — Fase aberta. Levantado o estado da simulação, os pedidos abertos e as restrições de gatilho; rascunhado o desenho com as decisões a tomar.
- 2026-11-18 — Etapa 5.1 aberta: roteiro do modo `stream` do simulador (para o João implementar) e Parte 1 do notebook `01_gold_orders_live` (tabelas, leitura em streaming, `reduce_batch`).
- 2026-11-17 — Decisões S1 a S8 aprovadas; ADRs 0009 a 0012 registrados.
- 2026-11-16 — Teste de consumo da gold no Power BI (fora do escopo da fase, registrado aqui): a conexão pelo conector do Databricks ao SQL warehouse funcionou no Free Edition, em modo Import; 28 medidas DAX reproduziram os KPIs oficiais (GMV R$ 15.614.632,11) e foi montado um relatório local de 6 páginas, uma por área de negócio. O relatório fica fora do repositório. Para a Fase 6: o teste importou tabelas com dado pessoal (cadastros e scorecard), o que a governança precisa decidir; e a separação de visão por área exige relatórios ou apps por público, já que a segurança por linha só filtra dados.

- 2026-11-18 — **Etapa 5.1 concluída.** O simulador ganhou o modo `stream`: emite **fatias de 6 horas** de um dia simulado (4 por dia), com a fatia no nome do arquivo (`<entidade>_AAAA-MM-DD_slice-NN_part-000.json`) na mesma pasta `sim_date=`; a fatia vem da hora do próprio evento e não vai para dentro do arquivo. O controle de progresso passou a ser por `(dia, fatia)` em `ops.simulator_runs` (`sim_slice_start`, `sim_slice_end`) e o gabarito `ops.simulator_chaos_log` ganhou `sim_slice`. Emitido o primeiro dia do stream, **2028-08-22 (4 fatias, 1.353 eventos de status)**, com falhas injetadas (duplicatas, 22 eventos atrasados na mesma fatia, fora de ordem). Resultado dos critérios de aceite no quadro abaixo.
- 2026-11-18 — Medições para a 5.2: cada fatia leva cerca de 1 minuto de notebook (48 a 59 s uma fatia; 137 s duas), fora a partida do job; emitir os 57 dias restantes (228 fatias) custaria cerca de 4 horas de serverless. Por isso o plano é emitir **sob demanda**, algumas fatias por vez, conforme cada etapa precisar de dados novos.

- 2026-11-18 — **Etapa 5.2 concluída.** A bronze continuou sem alteração. O notebook `01_gold_orders_live` ganhou a leitura em duas camadas (`read_events_clean`, sem deduplicação, e `read_events_stream`, com watermark), a opção `skipChangeCommits` e um teste da leitura (widget `teste_leitura`). Emitidos mais 3 dias do stream (2028-08-23 a 2028-08-25), com 3 ciclos de uma fatia cada e um ciclo de 9 fatias, todos pelo job de ciclo (simulador → bronze → leitura). Resultado no quadro abaixo.
- 2026-11-18 — **Etapa 5.3 concluída.** O notebook `01_gold_orders_live` ganhou a escrita: `foreachBatch` com `MERGE` por `event_sequence` maior (status, sequência e instante só avançam; instantes de etapa e data estimada só preenchem nulos; a linha só é reescrita se algo mudou), registro de cada micro-lote em `ops.streaming_batches` (eventos, pedidos, avanços, ignorados, tardios, latência) e a função `run_live`, usada na produção e nos testes. Carga inicial feita pelo próprio stream: 391.806 eventos viraram 99.148 pedidos em 71 s. O notebook da bronze ganhou o parâmetro `entidades` (caminho rápido). A medição do watermark mudou o desenho (ADR 0013). Emitidas as fatias 0 e 1 de 2028-08-26, em dois ciclos completos. Resultado no quadro abaixo.
- 2026-11-18 — **Etapa 5.4 concluída.** O notebook `01_gold_orders_live` ganhou a regra de risco (função pura, `RISK_CONFIG` em configuração), o recálculo do risco de **todos os pedidos abertos** ao fim de cada passada (`UPDATE` só nas linhas que mudam, porque "hoje" avança mesmo sem evento novo) e a view `olist.gold.vw_pedidos_em_risco`. Emitidas as fatias até 2028-08-27 (mais 6 fatias, em um ciclo). Para comparar o estado ao vivo com a gold batch em todos os pedidos, a bronze foi ingerida por completo e os pipelines `olist.silver` e `olist.gold` foram atualizados; a validação da Fase 4 (108 verificações) continuou sem reprovações. O novo notebook `src/quality/06_validacao_live` recalcula tudo por outro caminho em `quality.validacao_live`. Resultado no quadro abaixo.
- 2026-11-18 — **Etapa 5.5 concluída.** Dicionário de métricas v1.2 (seção "Visão ao vivo" e o que mudou da v1.1), notebook `src/quality/07_documentacao_live` (comentários e tags dos 5 objetos da fase, com conferência de coluna sem comentário), ADRs 0013 a 0015, limpeza das tabelas e checkpoints de prova, e fechamento do diário e do índice. A Fase 5 está concluída.

### Critérios de aceite da etapa 5.1

| # | Critério | Resultado |
|---|---|---|
| 1 | O modo `batch` gera os mesmos arquivos | ✅ 4 dias antigos regravados em área de prova (inclui a carga inicial, o valor fora do padrão e o arquivo corrompido): 27 arquivos com mesmo nome, tamanho e MD5 da landing real |
| 2 | Soma das fatias = dia de lote | ✅ nas 7 entidades diárias (ex.: eventos 1.353, pedidos 244) |
| 3 | Nenhuma linha em duas fatias | ✅ conferido por chave de registro, nas 7 entidades |
| 4 | Reprocessar uma fatia não duplica nem mexe nas outras | ✅ fatia 0 regravada com o mesmo MD5 e arquivos novos; fatia 1 intacta; nenhum arquivo a mais |
| 5 | Gabarito com `sim_slice` e `replaceWhere` por fatia | ✅ 65 linhas do dia, todas com fatia; reprocessar a fatia 0 manteve as da fatia 1 |
| 6 | O atraso mantém a fatia | ✅ 22 eventos atrasados: arquivo, gabarito e hora do evento apontam para a mesma fatia |
| 7 | A bronze, sem nenhuma alteração, ingere as fatias | ✅ após cada fatia entraram só as linhas novas (219, depois 519, depois 1.855); o arquivo regravado da fatia 0 não foi lido de novo |

### Critérios de aceite da etapa 5.2

| # | Critério | Resultado |
|---|---|---|
| 1 | Em 3 ciclos de uma fatia, a bronze recebe só as linhas da fatia nova | ✅ 89, 163 e 492 linhas de eventos, iguais ao que a fatia emitiu |
| 2 | A leitura em streaming lê só as linhas novas de cada ciclo; sem dado novo, lê 0 | ✅ leitura sem deduplicação: soma de todos os lotes = 413.108 linhas limpas da bronze (uma vez cada); o lote extra de cada passada vem vazio |
| 3 | Depois do filtro e da deduplicação, a contagem bate com as chaves distintas | ✅ 391.806 = chaves `(order_id, event_sequence)` distintas, inclusive no ciclo de 9 fatias, com atrasos de até 3 dias |
| 4 | Um `DELETE` na cópia da bronze não derruba a leitura com `skipChangeCommits` | ✅ DELETE ignorado, lidas só as 3 linhas inseridas. Sem a opção, a leitura falha com `DELTA_SOURCE_TABLE_IGNORE_CHANGES` (observado em duas execuções de prova) |
| 5 | Latência por etapa medida e comparada com o SLA de 15 min | ✅ ver abaixo |
| 6 | Nada alterado na bronze real nem na landing, além das fatias novas | ✅ bronze real com as mesmas linhas antes e depois do teste de robustez; cópia e checkpoint de prova apagados |

**Latência medida (ciclo de uma fatia, serverless):** o ciclo inteiro leva 8 a 9 minutos (simulador cerca de 3 min, bronze cerca de 4,5 min, leitura cerca de 1 min). Da gravação do arquivo na landing até a entrada na bronze, 50 a 62 s; até a leitura em streaming, 5 a 6 min. Isso cabe nos 15 min do SLA (ADR 0012), ainda sem o `MERGE` da 5.3. Um ciclo de 9 fatias levou 13 min (arquivo até a leitura, cerca de 10 min). O tempo da bronze vem de processar as 10 entidades em série (cerca de 28 s cada), mesmo as sem arquivo novo: para o monitor, ingerir só `order_status_events` e `orders` cortaria isso para cerca de 1 min (decisão para a 5.3 ou a Fase 7).

### Critérios de aceite da etapa 5.3

| # | Critério | Resultado |
|---|---|---|
| 1 | Uma linha por pedido, sem `order_id` repetido | ✅ 99.168 linhas, 99.168 pedidos (carga inicial mais dois ciclos) |
| 2 | Estado igual ao cálculo independente em SQL sobre a bronze | ✅ status, sequência, os 4 instantes de etapa e a data estimada iguais em todos os pedidos, depois da carga inicial e depois dos ciclos |
| 3 | `delivered` antes do `shipped` não perde o despacho | ✅ teste em tabelas de prova: pedido fica `delivered/4` com `shipped_at` preenchido; micro-lotes (eventos, avanços, ignorados, novos) = (2, 2, 0, 2), (4, 2, 1, 1), (1, 1, 0, 0) |
| 4 | Reprocessar não muda o estado | ✅ replay com checkpoint novo em outra tabela: mesmas linhas, pedidos e hash do estado (na prova e nos dados reais); reaplicar na mesma tabela também não muda nada |
| 5 | Evento tardio contado e aplicado | ✅ com o relógio de chegada (ADR 0013): evento distinto atrasado aplicado e contado (`late_events = 1`); com o `event_ts` como relógio, o Spark o descartou. Ver "Problemas encontrados" |
| 6 | Ciclos reais com a latência medida | ✅ arquivo na landing até a linha em `gold_orders_live`: 251 e 270 s (cerca de 4 a 4,5 min), dentro dos 15 min do SLA; ciclo completo de cerca de 7 min |
| 7 | Pré-conferência com `fact_orders` | ✅ 96.641 pedidos sem evento depois de 2028-08-21: status igual ao da gold batch em 100%. A validação formal em todos os pedidos é da etapa 5.4 |

**Tempos medidos:** carga inicial 71 s; bronze seletiva (2 entidades) cerca de 100 s, contra cerca de 280 s com as 10; `MERGE` de um micro-lote pequeno de 93 a 126 s (lê a tabela de estado e `bronze.orders` inteiras: candidato à otimização da Fase 8).

### Critérios de aceite da etapa 5.4

| # | Critério | Resultado |
|---|---|---|
| 1 | Regra de risco igual ao ADR 0011 | ✅ teste com 11 pedidos montados à mão (atrasado mesmo despachado, 0 e 2 dias em risco, 3 dias normal, despachado no prazo normal, fechados nulos) |
| 2 | Estado igual ao da silver e da gold batch em **todos** os pedidos | ✅ 99.291 pedidos: status, sequência, instantes de etapa e data estimada iguais à silver; status e data de entrega iguais a `fact_orders` |
| 3 | Nenhum evento perdido | ✅ 392.261 eventos distintos da bronze = soma dos eventos que chegaram ao `MERGE` |
| 4 | Latência dentro do SLA | ✅ 3 micro-lotes incrementais, o maior com 503 s (8,4 min), contra 900 s |
| 5 | Lista de risco igual a um cálculo independente | ✅ nível de risco igual em todos os pedidos; nenhum pedido fechado com risco, nenhum aberto sem risco; a view lista exatamente os 1.847 atrasados e em risco, sem duplicata |
| 6 | Idempotência | ✅ replay com checkpoint novo e execução sem arquivo novo dão o mesmo hash do estado, recalculado em SQL pela validação |
| 7 | Tardio contado | ✅ `late_events` registrado por micro-lote (0 nos dados reais, 1 no teste do watermark) |
| 8 | Eventos órfãos | ✅ nenhum evento de pedido que não existe na silver |

`quality.validacao_live`: 27 verificações, 21 com resultado esperado × obtido (todas OK) e 6 só de registro (contagens por nível, latência máxima, número de micro-lotes e tardios).

**Risco ao longo do tempo:** na data simulada 2028-08-26 havia 1.831 pedidos `atrasado`, 12 `em_risco` e 1.406 `normal`; em 2028-08-27, 1.822, 25 e 1.439. Dos 1.822 atrasados, 1.466 (80%) estão atrasados há mais de 90 dias e só 104 há até 7 dias: a maior parte é histórico que nunca foi entregue nos dados de origem (ver dicionário v1.2).

**KPIs oficiais, antes e depois de atualizar os pipelines com 22 a 27/08:** pedidos criados 98.476 → 99.291; válidos 97.265 → 98.079; GMV R$ 15.614.632,11 → R$ 15.723.094,10; entregues 93.247 → 94.804; entregas no prazo 93,27% → 93,27%.

### Critérios de aceite da etapa 5.5

| # | Critério | Resultado |
|---|---|---|
| 1 | Dicionário v1.2 versionado | ✅ seção "Visão ao vivo" e "O que mudou da v1.1 para a v1.2" |
| 2 | Objetos documentados e com tags | ✅ 5 objetos com `camada`, `tipo_tabela` e `contem_pii`, comentário de tabela e nenhuma coluna sem comentário |
| 3 | ADRs das decisões da fase | ✅ 0013 (relógio do watermark), 0014 (caminho rápido da bronze) e 0015 (`skipChangeCommits`) |
| 4 | Objetos de prova removidos | ✅ tabelas `ops.teste_*` e checkpoints de prova apagados; ficam `ops.streaming_batches` e `ops.streaming_conferencias` |
| 5 | Diário e índice fechados; `docs/` sem URL, e-mail ou comando | ✅ |

## Decisões

Aprovadas em 2026-11-17, todas na opção recomendada (S1 a S8 no [desenho](../design/fase-5-streaming.md)):

- [ADR 0009](../adr/0009-simulador-em-modo-stream-e-relogio-fino.md) — simulador em modo `stream` com fatias de horas; "hoje" é a data simulada.
- [ADR 0010](../adr/0010-gold-orders-live-merge-idempotente-e-watermark.md) — `foreachBatch` + `MERGE` por `event_sequence` maior; `dropDuplicatesWithinWatermark` com 4 dias.
- [ADR 0011](../adr/0011-regra-de-pedido-em-risco.md) — `atrasado` e `em_risco` (N = 2 dias), só pedidos abertos.
- [ADR 0012](../adr/0012-sla-do-monitor-e-numero-oficial-versus-ao-vivo.md) — SLA de 15 min; batch é o oficial, ao vivo é operacional.

Decididas durante a implementação (2026-11-18):

- [ADR 0013](../adr/0013-relogio-do-watermark-instante-de-chegada.md) — o watermark conta pelo instante de chegada à bronze, não pelo `event_ts`; substitui os itens 4 e 5 do ADR 0010.
- [ADR 0014](../adr/0014-caminho-rapido-da-bronze-ingestao-seletiva.md) — parâmetro `entidades` na bronze e ciclo que ingere só as duas entidades do monitor.
- [ADR 0015](../adr/0015-leitura-em-streaming-da-bronze-com-skipchangecommits.md) — a leitura ignora commits que reescrevem a bronze; a exclusão de cliente da Fase 6 não chega sozinha ao estado ao vivo.

## Problemas encontrados

- **Pasta de preparação com resto da fatia anterior (etapa 5.1).** O primeiro teste da escrita por fatia falhou com `sellers 2028-08-22: esperado 1 arquivo na preparação, encontrados 0`. O `sellers` tinha 1 registro na fatia 0 e nenhum na fatia 1: depois de publicar a fatia 0, sobrava a pasta `sim_date=` vazia na preparação, e a escrita vazia da fatia 1 não a limpava. Correção: a preparação de cada entidade começa sempre vazia. Só aparece em fatia onde uma entidade fica sem registro, e o teste cobriu o caso.
- **O Spark descartava o evento atrasado (etapa 5.3).** O ADR 0010 supunha que um evento mais velho que o watermark, pelo `event_ts`, seria aplicado pelo `MERGE` e contado. A medição em tabelas de prova mostrou o contrário: nem o evento distinto nem os duplicados tardios chegaram ao `MERGE`, e o pedido ficou no status anterior, sem nenhum registro do descarte. Nos dados reais a margem de 4 dias sobre atraso de 3 dias escondia o problema. Correção: o relógio do watermark passou a ser o instante de chegada (`_ingested_at`, ADR 0013); o teste `watermark` roda os dois relógios lado a lado e fica como evidência repetível.
- **Stream que falha de propósito derruba o comando (etapa 5.2).** Para provar que a leitura sem `skipChangeCommits` falha depois de um `DELETE`, o teste capturava o erro com `try/except`, mas o Databricks encerra o comando com "Some streams terminated before this command could finish" quando qualquer stream termina com erro, mesmo tratado (`resetTerminated` não resolveu). O caso negativo foi observado em duas execuções de prova e ficou só documentado; o teste automatizado afirma o caso positivo.
- **Retentativa automática dobrou medições (etapa 5.2).** Uma task do job one-time que falhou foi repetida sozinha e gravou as medições duas vezes. As tasks do ciclo passaram a usar `max_retries = 0`, e o teste da robustez recomeça as próprias medições.
- **Células de controle executadas por engano (etapa 5.1).** Durante os testes, células de controle do modo `batch` deixaram uma linha `RUNNING` órfã em `ops.simulator_runs`; foi fechada como `FAILED` com mensagem explicativa. Por isso a trava temporária do modo `stream` só saiu quando o fluxo por fatia estava completo.
- **Validação que não executou nada não valida.** A primeira versão da idempotência da etapa 5.4 chamava outro notebook com `dbutils.notebook.run`; o Free Edition permite uma execução ativa por vez e a chamada falhou, mas o "antes e depois" passou porque nada tinha mudado. A verificação foi refeita para ler o registro da conferência do monitor (`ops.streaming_conferencias`), exigir que ela seja mais nova que o último micro-lote e comparar o hash recalculado em SQL. As linhas dessa execução inválida foram apagadas de `quality.validacao_live`.
- **A lista de atrasados é dominada por pedidos antigos (etapa 5.4).** 80% dos pedidos `atrasado` estão atrasados há mais de 90 dias: são pedidos do histórico que nunca foram entregues nos dados de origem. A regra do ADR 0011 foi mantida; o dicionário orienta filtrar por `dias_para_a_entrega`. Uma janela de ação (por exemplo, só atrasos recentes) fica como melhoria.

## Aprendizados

- **A pasta é o contrato com o Auto Loader; o nome do arquivo é livre.** Manter `sim_date=AAAA-MM-DD/` e acrescentar a fatia só no nome do arquivo fez a bronze funcionar sem nenhuma alteração: o relógio fino do simulador não custou mudança no consumidor.
- **Reprocessar é seguro porque o arquivo é determinístico.** A fatia regravada saiu byte a byte igual (mesmo MD5) e, como o Auto Loader não relê o mesmo caminho (`allowOverwrites = false`), a bronze não duplicou.
- **Testar na área de prova antes de gravar de verdade.** O defeito da pasta de preparação foi achado e corrigido sem sujar a landing nem as tabelas de controle.
- **Limitação conhecida:** `horas_por_fatia` precisa ser o mesmo em todas as execuções do stream; a tabela de controle guarda a fatia, não o tamanho dela. Se uma execução falhar no meio de várias fatias, as anteriores ficam publicadas e a próxima execução recomeça da última com `SUCCESS` (reescrever é seguro).
- **O checkpoint é o contador de "linhas já lidas".** Cada leitura de prova tem checkpoint próprio, então a mesma bronze pode ser lida várias vezes sem interferir; apagar o checkpoint faz o stream reler tudo (cerca de 410 mil linhas em menos de 1 minuto).
- **Fonte Delta em streaming só aceita `INSERT` por padrão.** A exclusão de cliente da Fase 6 vai alterar a bronze; a leitura já nasce com `skipChangeCommits`, ao custo de essas exclusões não chegarem sozinhas a `gold_orders_live` (a Fase 6 precisa tratá-las).
- **A deduplicação não perdeu nenhuma chave nos testes**, mas todos os atrasos ficaram dentro do mesmo lote ou do watermark; o caso de um duplicado que chega depois do watermark continua para a 5.3, onde o `MERGE` por sequência é a segunda defesa.
- **Medir antes de confiar na documentação ou na suposição.** A premissa do ADR 0010 sobre o watermark só caiu porque a etapa pediu um teste com evento mais velho que o limite; o código de produção, sozinho, teria passado em todos os testes de dados reais.
- **A mesma função roda na produção e nos testes.** Passar tabelas e checkpoint como parâmetro de `run_live` evitou duplicar a lógica e permitiu o replay com checkpoint novo como prova de idempotência.
- **`MERGE` condicional preserva a identidade da linha.** Só reescrever a linha quando algo muda mantém `_updated_at` estável e deixa o hash do estado comparável entre execuções.

## Encerramento

Todos os critérios do desenho foram atendidos (resultado detalhado nas tabelas de cada etapa): o monitor mantém **uma linha por pedido** em `gold_orders_live` (99.291 pedidos), com status igual ao da gold batch em 100% dos pedidos e igual a um cálculo independente da silver; duplicatas, atrasos e eventos fora de ordem não mudam o estado; reprocessar com checkpoint novo dá o mesmo hash; nenhum evento é perdido (o relógio do watermark foi corrigido pelo ADR 0013); a latência medida do arquivo na landing até a linha ao vivo foi de 4 a 8,4 minutos, dentro dos 15 do SLA; a lista de pedidos em risco é igual a um cálculo independente; o dicionário v1.2, as tags e os ADRs 0009 a 0015 registram o que foi decidido.

Números da fase: simulador em modo `stream` com fatias de 6 h (4 por dia), 6 dias emitidos (2028-08-22 a 2028-08-27); carga inicial de 391.806 eventos em 71 s; ciclo completo (simulador, bronze seletiva e streaming) de cerca de 7 minutos; 27 verificações em `quality.validacao_live`.

Pendências para as próximas fases:

- **Dias restantes do stream:** 52 dias (2028-08-28 a 2028-10-18, 208 fatias) não foram emitidos: custariam cerca de 3 horas de serverless. A capacidade está provada; o mesmo job de ciclo os emite em lotes quando fizer sentido (por exemplo, para a demonstração final).
- **Fase 6 (governança e LGPD):** a exclusão de cliente apaga linhas da bronze e **não chega sozinha** a `gold_orders_live` (ADR 0015); o procedimento de exclusão deve reconstruir o estado ao vivo ou apagar o pedido nele. Os objetos ao vivo não têm dado pessoal (`contem_pii = false`).
- **Fase 7 (testes, CI/CD, orquestração):** agendar o ciclo (simulador, bronze seletiva, streaming) com retentativas controladas; hoje roda sob demanda. Testes unitários de `reduce_batch`, `risk_sql` e `merge_into_live`; módulo único para as funções que hoje se repetem entre notebooks. O Free Edition permite uma execução ativa por vez: o orquestrador precisa de tasks em sequência, não de notebooks aninhados.
- **Fase 8 (performance):** o `MERGE` de um micro-lote pequeno leva de 1,5 a 2 minutos (lê a tabela de estado e `bronze.orders` inteiras); candidato a leitura por chave. A ingestão seletiva (ADR 0014) já corta a bronze de cerca de 280 para 100 s.
- **Melhoria da lista de risco:** janela de ação (atrasos recentes) e, se pedido, o refinamento por lead time mínimo que o ADR 0011 deixou para uma versão seguinte.
- **Dashboard e Genie:** incluir a lista de pedidos em risco, rotulada como operacional (ADR 0012).
