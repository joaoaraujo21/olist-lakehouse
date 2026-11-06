# Fase 1 — Simulador e Bronze

- **Período previsto:** semanas 2–3
- **Status:** 🔄 em andamento — etapa 1.1 (desenho)
- **Branch/PRs:** `fase-1/desenho` — PR #3 (desenho técnico + diário de progresso)

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
| 1.2 Simulador base | lotes "limpos" em `raw_data/<entidade>/` + `ops.simulator_runs` | — |
| 1.3 Bronze | Auto Loader por entidade + auditoria | — |
| 1.4 Caos + CDC | duplicatas, atraso, fora de ordem, schema, CDC | — |
| 1.5 Aceite | 3× sem duplicar; schema quebrado e stream sobrevive; ADRs | — |

## O que foi feito

- 2026-11-02 — Etapa 1.1: perguntas de desenho levantadas (A1–A5 simulador, B1–B6 bronze) e respondidas no documento de desenho técnico, com contexto de negócio, escopo, justificativas, alternativas, riscos e critérios de aceite. Estudo da documentação oficial (Auto Loader, schema, `_metadata`, triggers, checkpoints, limites do serverless).
- 2026-11-06 — Desenho e diário de progresso publicados no PR #3 (`fase-1/desenho`).
- 2026-11-06 — PR #3 mergeado (como merge commit, não squash: o commit `dbdf3cd`, com URL do workspace e e-mail, ficou no histórico da `main`; decidido não reescrever o histórico por ser dado de baixa sensibilidade). Branches antigas apagadas.
- 2026-11-06 — Etapa 1.2a especificada: volume `ops.checkpoints`, tabelas `ops.simulator_runs` (com CHECK constraint em `status`) e `ops.simulator_chaos_log`, com COMMENT em tabelas e colunas; branch `fase-1/estrutura-controle`. Colunas do `chaos_log` padronizadas em inglês no desenho (A5).
- **Próximo passo:** João implementa a 1.2a no notebook de setup → PR → review. Depois, 1.2b (notebook do simulador, sem caos).

## Decisões

Ver o resumo no fim de `docs/design/fase-1-simulador-bronze.md`. Principais: datas +3.654 dias (522 semanas); relógio diário com `dias_por_execucao`; estado em `ops.simulator_runs`; `orders` dividido em cabeçalho + `order_status_events`; `customers`/`sellers` como CDC; JSON Lines com caminho determinístico `raw_data/<entidade>/sim_date=.../`; caos determinístico com gabarito em `ops.simulator_chaos_log`; bronze em notebook (Structured Streaming + Auto Loader), `availableNow`, checkpoints em `/Volumes/olist/ops/checkpoints/bronze/<entidade>/`, tudo string + `addNewColumns`; dedup de negócio na silver.

## Problemas encontrados e como foram resolvidos

_(preencher durante a fase)_

## Aprendizados

_(preencher durante a fase)_

## Encerramento — pendências para a Fase 2

_(preencher ao concluir)_
