# Fase 5 — Streaming

- **Período:** iniciada em 2026-11-16
- **Status:** 🟡 em andamento — [desenho técnico](../design/fase-5-streaming.md) aprovado (S1 a S8); etapa 5.1 (simulador em modo `stream`) concluída, a iniciar a 5.2
- **Branch/PRs:** `fase-5` (etapa 5.1); as etapas seguintes saem em `fase-5/<nome>`, uma por PR

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
| 5.2 | Ingestão dos novos eventos na bronze e leitura em streaming (a bronze já ingere as fatias sem alteração: resta o agendamento e a leitura em streaming) | a iniciar |
| 5.3 | `gold_orders_live`: deduplicação com watermark e `MERGE` idempotente em `foreachBatch` | a iniciar |
| 5.4 | Regra de risco de atraso, lista de pedidos em risco e validação independente | a iniciar |
| 5.5 | Documentação, ADRs e encerramento | a iniciar |

## O que foi feito

- 2026-11-16 — Fase aberta. Levantado o estado da simulação, os pedidos abertos e as restrições de gatilho; rascunhado o desenho com as decisões a tomar.
- 2026-11-18 — Etapa 5.1 aberta: roteiro do modo `stream` do simulador (para o João implementar) e Parte 1 do notebook `01_gold_orders_live` (tabelas, leitura em streaming, `reduce_batch`).
- 2026-11-17 — Decisões S1 a S8 aprovadas; ADRs 0009 a 0012 registrados.
- 2026-11-16 — Teste de consumo da gold no Power BI (fora do escopo da fase, registrado aqui): a conexão pelo conector do Databricks ao SQL warehouse funcionou no Free Edition, em modo Import; 28 medidas DAX reproduziram os KPIs oficiais (GMV R$ 15.614.632,11) e foi montado um relatório local de 6 páginas, uma por área de negócio. O relatório fica fora do repositório. Para a Fase 6: o teste importou tabelas com dado pessoal (cadastros e scorecard), o que a governança precisa decidir; e a separação de visão por área exige relatórios ou apps por público, já que a segurança por linha só filtra dados.

- 2026-11-18 — **Etapa 5.1 concluída.** O simulador ganhou o modo `stream`: emite **fatias de 6 horas** de um dia simulado (4 por dia), com a fatia no nome do arquivo (`<entidade>_AAAA-MM-DD_slice-NN_part-000.json`) na mesma pasta `sim_date=`; a fatia vem da hora do próprio evento e não vai para dentro do arquivo. O controle de progresso passou a ser por `(dia, fatia)` em `ops.simulator_runs` (`sim_slice_start`, `sim_slice_end`) e o gabarito `ops.simulator_chaos_log` ganhou `sim_slice`. Emitido o primeiro dia do stream, **2028-08-22 (4 fatias, 1.353 eventos de status)**, com falhas injetadas (duplicatas, 22 eventos atrasados na mesma fatia, fora de ordem). Resultado dos critérios de aceite no quadro abaixo.
- 2026-11-18 — Medições para a 5.2: cada fatia leva cerca de 1 minuto de notebook (48 a 59 s uma fatia; 137 s duas), fora a partida do job; emitir os 57 dias restantes (228 fatias) custaria cerca de 4 horas de serverless. Por isso o plano é emitir **sob demanda**, algumas fatias por vez, conforme cada etapa precisar de dados novos.

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

## Decisões

Aprovadas em 2026-11-17, todas na opção recomendada (S1 a S8 no [desenho](../design/fase-5-streaming.md)):

- [ADR 0009](../adr/0009-simulador-em-modo-stream-e-relogio-fino.md) — simulador em modo `stream` com fatias de horas; "hoje" é a data simulada.
- [ADR 0010](../adr/0010-gold-orders-live-merge-idempotente-e-watermark.md) — `foreachBatch` + `MERGE` por `event_sequence` maior; `dropDuplicatesWithinWatermark` com 4 dias.
- [ADR 0011](../adr/0011-regra-de-pedido-em-risco.md) — `atrasado` e `em_risco` (N = 2 dias), só pedidos abertos.
- [ADR 0012](../adr/0012-sla-do-monitor-e-numero-oficial-versus-ao-vivo.md) — SLA de 15 min; batch é o oficial, ao vivo é operacional.

## Problemas encontrados

- **Pasta de preparação com resto da fatia anterior (etapa 5.1).** O primeiro teste da escrita por fatia falhou com `sellers 2028-08-22: esperado 1 arquivo na preparação, encontrados 0`. O `sellers` tinha 1 registro na fatia 0 e nenhum na fatia 1: depois de publicar a fatia 0, sobrava a pasta `sim_date=` vazia na preparação, e a escrita vazia da fatia 1 não a limpava. Correção: a preparação de cada entidade começa sempre vazia. Só aparece em fatia onde uma entidade fica sem registro, e o teste cobriu o caso.
- **Células de controle executadas por engano (etapa 5.1).** Durante os testes, células de controle do modo `batch` deixaram uma linha `RUNNING` órfã em `ops.simulator_runs`; foi fechada como `FAILED` com mensagem explicativa. Por isso a trava temporária do modo `stream` só saiu quando o fluxo por fatia estava completo.

## Aprendizados

- **A pasta é o contrato com o Auto Loader; o nome do arquivo é livre.** Manter `sim_date=AAAA-MM-DD/` e acrescentar a fatia só no nome do arquivo fez a bronze funcionar sem nenhuma alteração: o relógio fino do simulador não custou mudança no consumidor.
- **Reprocessar é seguro porque o arquivo é determinístico.** A fatia regravada saiu byte a byte igual (mesmo MD5) e, como o Auto Loader não relê o mesmo caminho (`allowOverwrites = false`), a bronze não duplicou.
- **Testar na área de prova antes de gravar de verdade.** O defeito da pasta de preparação foi achado e corrigido sem sujar a landing nem as tabelas de controle.
- **Limitação conhecida:** `horas_por_fatia` precisa ser o mesmo em todas as execuções do stream; a tabela de controle guarda a fatia, não o tamanho dela. Se uma execução falhar no meio de várias fatias, as anteriores ficam publicadas e a próxima execução recomeça da última com `SUCCESS` (reescrever é seguro).

## Encerramento

A preencher.
