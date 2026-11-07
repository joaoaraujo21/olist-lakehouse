# Fase 1 — Simulador e Bronze

- **Período previsto:** semanas 2–3
- **Status:** 🔄 em andamento — etapa 1.2b concluída; próxima: 1.3 (bronze)
- **Branch/PRs:** `fase-1/desenho` (PR #3), `fase-1/estrutura-controle` (PR #4), `fase-1/simulador-base` (PR do simulador base)

## Objetivo

**Negócio:** a operação "acontece" todo dia; precisamos capturar tudo sem perder dado.

**Técnico:**
1. Simulador Python lendo de `olist.landing.olist_source` e gravando lotes em `olist.landing.raw_data/<entidade>/`: pedidos reencenados "no presente", com duplicatas, atrasos e eventos fora de ordem injetados, mais um feed de mudanças cadastrais (vendedores/clientes).
2. Ingestão bronze com Auto Loader: Delta append-only, colunas de auditoria (`_ingested_at`, `_source_file`, `_run_id`), `rescuedDataColumn`, evolução de schema.

**Entrega:** bronze completa, incremental e idempotente.

**Desafios de aceite:**
- Rodar o gerador 3 vezes e provar que não há duplicidade indevida na bronze.
- Quebrar o schema de propósito e o pipeline sobreviver.

**Fluxo da fase:** contexto de negócio e requisitos → desenho técnico aprovado antes de codar → implementação na interface → code review → ADR da principal decisão → PR em branch `fase-1/...`.

## Etapas

| Etapa | Entrega | Status |
|---|---|---|
| 1.1 Desenho | `docs/design/fase-1-simulador-bronze.md` | ✅ concluída — PR #3 |
| 1.2a Estrutura de controle | volume `ops.checkpoints`, `ops.simulator_runs`, `ops.simulator_chaos_log` | ✅ concluída — PR #4 |
| 1.2b Simulador base | notebook `src/simulador/01_simulador_base`: lotes "limpos" em `raw_data/<entidade>/` | ✅ concluída — PR do simulador base |
| 1.3 Bronze | Auto Loader por entidade + auditoria | — |
| 1.4 Caos + CDC | duplicatas, atraso, fora de ordem, schema, CDC | — |
| 1.5 Aceite | 3× sem duplicar; schema quebrado e stream sobrevive; ADRs | — |

## O que foi feito

- 2026-11-02 — Etapa 1.1: perguntas de desenho levantadas (A1–A5 simulador, B1–B6 bronze) e respondidas no documento de desenho técnico, com contexto de negócio, escopo, justificativas, alternativas, riscos e critérios de aceite. Estudo da documentação oficial (Auto Loader, schema, `_metadata`, triggers, checkpoints, limites do serverless).
- 2026-11-06 — Desenho e diário de progresso publicados no PR #3 (`fase-1/desenho`).
- 2026-11-06 — PR #3 mergeado (como merge commit, não squash: o commit `dbdf3cd`, com URL do workspace e e-mail, ficou no histórico da `main`; decidido não reescrever o histórico por ser dado de baixa sensibilidade). Branches antigas apagadas.
- 2026-11-06 — Etapa 1.2a especificada: volume `ops.checkpoints`, tabelas `ops.simulator_runs` (com CHECK constraint em `status`) e `ops.simulator_chaos_log`, com COMMENT em tabelas e colunas; branch `fase-1/estrutura-controle`. Colunas do `chaos_log` padronizadas em inglês no desenho (A5).
- 2026-11-06 — Etapa 1.2a implementada (PR #4, branch `fase-1/estrutura-controle`): volume `ops.checkpoints`, tabelas `ops.simulator_runs` (NOT NULL + CHECK `status_valido`) e `ops.simulator_chaos_log`, comentários em tabelas e colunas, comentário do schema `ops` atualizado via `COMMENT ON`. Conferido no Databricks. Ajustes de formatação do notebook concluídos; títulos de células descartados (seções em Markdown já organizam o notebook).
- 2026-11-06 — PR #4 mergeado com squash. Etapa 1.2b especificada: notebook `src/simulador/01_simulador_base` em 6 partes (configuração/widgets, leitura + deslocamento de datas, montagem das 10 entidades com `sim_date`, controle da execução, escrita determinística via staging + renomeação, finalização SUCCESS/FAILED). Decisão de implementação: Spark grava em `_staging/<run_id>/` com `partitionBy("sim_date")` e o código move cada `part-*.json` para o nome definitivo, porque o Spark não permite escolher o nome do arquivo.
- 2026-11-06 — Etapa 1.2b, Parte 1 (configuração e widgets) implementada no notebook `src/simulador/01_simulador_base`, branch `fase-1/simulador-base`: widgets `dias_por_execucao`, `sim_date_override` e `run_id`, constantes de caminho derivadas do catálogo, validação dos parâmetros em função pura (`parse_params`). Environment versão 6, sem dependências. Parte 2 (leitura e deslocamento de datas) especificada: deslocamento via `TIMESTAMP_NTZ`, para que o fuso da sessão e o antigo horário de verão brasileiro (vigente em 2016–2018) não alterem os horários.
- 2026-11-06 — Etapa 1.2b, Parte 2 implementada e validada: deslocamento de exatamente 3.654 dias em todos os pedidos, nenhum com dia da semana alterado, primeira compra em 2026-09-06 e última em 2028-10-18 (critério de aceite nº 5).
- 2026-11-06 — Etapa 1.2b, Parte 3 especificada: montagem das 10 entidades com `sim_date` (funções puras por entidade). Eventos de status via `stack` (sequências fixas 1–4) mais evento final (sequência 5) para `canceled`, `unavailable`, `invoiced` e `processing`, datado com `greatest` das datas do pedido; total esperado de 394.705 eventos. Nomes das entidades da Parte 2 alinhados ao desenho (`order_payments`, `order_reviews`).
- 2026-11-06 — Parte 3 validada: contagens das 9 entidades de origem preservadas após os joins; 394.705 eventos de status (99.441 created, 99.281 approved, 97.658 shipped, 96.476 delivered, 1.849 finais); pedido entregue com 4 eventos nos respectivos dias (critério de aceite nº 6).
- 2026-11-06 — Parte 4 especificada: `SIM_END` = dia da última compra (2028-10-18); janela calculada em função pura (`compute_window`), com reprocessamento permitido só para dias já gerados ou o próximo (sem lacunas no progresso); registro `RUNNING` via `MERGE` por `run_id` (retentativa do job com o mesmo `run_id` reabre a linha em vez de duplicar); valores passados por parameter markers.
- 2026-11-06 — Parte 4 validada: janela calculada, `MERGE` idempotente com `run_id` fixo, bloqueio de override que pularia dias. Linhas de teste removidas de `ops.simulator_runs`.
- 2026-11-06 — Parte 5 especificada: escrita em duas fases — preparação de todas as entidades em `_staging/<run_id>/` (`repartition` + `sortWithinPartitions` + `partitionBy("sim_date")`, `ignoreNullFields=false` para o schema não depender de valores nulos) e, só então, limpeza dos dias da janela no destino e publicação com nome definitivo; pasta de preparação apagada em `finally`. Entidades de carga inicial só são lidas quando a janela inclui `SIM_START`.
- 2026-11-07 — Parte 5 validada no volume: um arquivo por entidade em `sim_date=2026-09-06/` com nome determinístico, preparação vazia, strings preservadas (CEP com zero à esquerda). O primeiro dia tem 1 pedido; `geolocation` (carga inicial) gera o maior arquivo, com cerca de 176 MB.
- 2026-11-07 — Parte 6 especificada: execução em `try/except/finally` — `SUCCESS` com arquivos e linhas, `FAILED` com a mensagem de erro e relançamento da exceção, limpeza da preparação sempre. A célula de execução da Parte 5 passou para a Parte 6. Roteiro de aceite com verificação de MD5 em notebook auxiliar fora do repositório.
- 2026-11-07 — Aceite da 1.2b: 1ª execução gerou 2026-09-06 com `SUCCESS` (9 arquivos, 1.036.286 linhas, quase todas da carga inicial: 1.000.163 de `geolocation`); 2ª execução avançou sozinha para 2026-09-07 (5 arquivos, 5 linhas — entidades de carga inicial não lidas); preparação vazia. 3ª execução com `sim_date_override = 2026-09-06` regravou os 9 arquivos do dia com MD5 idêntico ao original, e o progresso permaneceu em 2026-09-07 (critério nº 3). Lição registrada: a primeira tentativa do teste de MD5 comparou arquivos que não tinham sido regravados (o reprocessamento não chegou a rodar) — a evidência do teste (nova linha em `simulator_runs` e data de modificação do arquivo) precisa ser conferida, não só o resultado "OK".
- **Próximo passo:** limpar outputs → commit + PR da branch `fase-1/simulador-base` (com os arquivos de `docs/progresso/` atualizados) → review → etapa 1.3 (bronze).

## Decisões

Ver o resumo no fim de `docs/design/fase-1-simulador-bronze.md`. Principais: datas +3.654 dias (522 semanas); relógio diário com `dias_por_execucao`; estado em `ops.simulator_runs`; `orders` dividido em cabeçalho + `order_status_events`; `customers`/`sellers` como CDC; JSON Lines com caminho determinístico `raw_data/<entidade>/sim_date=.../`; caos determinístico com gabarito em `ops.simulator_chaos_log`; bronze em notebook (Structured Streaming + Auto Loader), `availableNow`, checkpoints em `/Volumes/olist/ops/checkpoints/bronze/<entidade>/`, tudo string + `addNewColumns`; dedup de negócio na silver.

## Problemas encontrados e como foram resolvidos

_(preencher durante a fase)_

## Aprendizados

_(preencher durante a fase)_

## Encerramento — pendências para a Fase 2

_(preencher ao concluir)_
