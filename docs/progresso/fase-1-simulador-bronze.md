# Fase 1 — Simulador e Bronze

- **Período previsto:** semanas 2–3
- **Status:** 🔄 em andamento — etapa 1.1 (desenho)
- **Branch/PRs:** `fase-1/desenho` (a criar)

## Objetivo

**Negócio:** a operação "acontece" todo dia; precisamos capturar tudo sem perder dado.

**Técnico:**
1. Simulador Python lendo de `olist.landing.olist_source` e gravando lotes em `olist.landing.raw_data/<entidade>/`: pedidos reencenados "no presente", com duplicatas, atrasos e eventos fora de ordem injetados, mais um feed de mudanças cadastrais (vendedores/clientes).
2. Ingestão bronze com Auto Loader: Delta append-only, colunas de auditoria (`_ingested_at`, `_source_file`, `_run_id`), `rescuedDataColumn`, evolução de schema.

**Entrega:** bronze completa, incremental e idempotente.

**Desafios de aceite:**
- Rodar o gerador 3 vezes e provar que não há duplicidade indevida na bronze.
- Quebrar o schema de propósito e o pipeline sobreviver.

**Fluxo combinado:** Claude apresenta o desafio e os requisitos → João desenha a solução antes de codar → implementação na interface → code review → ADR da principal decisão → PR em branch `fase-1/...`.

## Etapas

| Etapa | Entrega | Status |
|---|---|---|
| 1.1 Desenho | `docs/design/fase-1-simulador-bronze.md` | rascunho pronto (Claude, a pedido do João); falta revisar e commitar |
| 1.2 Simulador base | lotes "limpos" em `raw_data/<entidade>/` + `ops.simulator_runs` | — |
| 1.3 Bronze | Auto Loader por entidade + auditoria | — |
| 1.4 Caos + CDC | duplicatas, atraso, fora de ordem, schema, CDC | — |
| 1.5 Aceite | 3× sem duplicar; schema quebrado e stream sobrevive; ADRs | — |

## O que foi feito

- 2026-11-02 — Desafio da etapa 1.1 apresentado (perguntas A1–A5 e B1–B6). O João pediu as sugestões de melhor prática; o documento de desenho foi escrito com pergunta, resposta, porquê e alternativas.
- 2026-11-02 — Documento reescrito no formato "Pergunta / Resposta" (uma pergunta por subitem), em tom de documento técnico formal (responsável técnico: João), com contexto de negócio, escopo, riscos e critérios de aceite. João vai ler o documento e a documentação oficial antes de seguir.
- **Próximo passo ao retomar:** apagar a branch `fase-0/setup` → criar `fase-1/desenho` → commitar `docs/design/fase-1-simulador-bronze.md` (+ `progresso/`) → PR para revisão → depois, etapa 1.2 começando pela inclusão do volume `olist.ops.checkpoints` e das tabelas `ops.simulator_runs`/`ops.simulator_chaos_log` no setup.

## Decisões

Ver o resumo no fim de `docs/design/fase-1-simulador-bronze.md`. Principais: datas +3.654 dias (522 semanas); relógio diário com `dias_por_execucao`; estado em `ops.simulator_runs`; `orders` dividido em cabeçalho + `order_status_events`; `customers`/`sellers` como CDC; JSON Lines com caminho determinístico `raw_data/<entidade>/sim_date=.../`; caos determinístico com gabarito em `ops.simulator_chaos_log`; bronze em notebook (Structured Streaming + Auto Loader), `availableNow`, checkpoints em `/Volumes/olist/ops/checkpoints/bronze/<entidade>/`, tudo string + `addNewColumns`; dedup de negócio na silver.

## Problemas encontrados e como foram resolvidos

_(preencher durante a fase)_

## Aprendizados

_(preencher durante a fase)_

## Encerramento — pendências para a Fase 2

_(preencher ao concluir)_
