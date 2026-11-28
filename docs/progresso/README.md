# Progresso do projeto — Olist Lakehouse

Índice do diário de bordo. Cada fase tem seu próprio arquivo com o que foi feito, as decisões e as pendências; este arquivo guarda só o que vale para o projeto todo.

**Última atualização:** 2026-11-28 · **Fase atual:** Fase 6 — Governança e LGPD (etapas 6.1 a 6.4 executadas; 6.4 aguarda commit e PR; próxima: 6.5, auditoria e encerramento)

## Fases

| Fase | Status | Registro |
|---|---|---|
| 0 — Fundação | ✅ concluída em 2026-11-02 | [fase-0-fundacao.md](fase-0-fundacao.md) |
| 1 — Simulador e Bronze | ✅ concluída em 2026-11-08 | [fase-1-simulador-bronze.md](fase-1-simulador-bronze.md) |
| 2 — Silver e Qualidade | ✅ concluída em 2026-11-12 | [fase-2-silver-qualidade.md](fase-2-silver-qualidade.md) |
| 3 — CDC e SCD2 | ✅ concluída em 2026-11-13 | [fase-3-cdc-scd2.md](fase-3-cdc-scd2.md) |
| 4 — Gold e Métricas | ✅ concluída em 2026-11-16 | [fase-4-gold-metricas.md](fase-4-gold-metricas.md) |
| 5 — Streaming | ✅ concluída em 2026-11-18 | [fase-5-streaming.md](fase-5-streaming.md) |
| 6 — Governança e LGPD | 🟡 em andamento | [fase-6-governanca-lgpd.md](fase-6-governanca-lgpd.md) |
| 7 — Testes, CI/CD e orquestração | — | |
| 8 — Performance e revisão final | — | |
| 9 — ML (opcional) | — | |

Ao concluir uma fase: marcar ✅ aqui, fechar o arquivo da fase (seção "Encerramento") e criar o arquivo da fase seguinte.

## Onde as coisas estão

| O quê | Onde |
|---|---|
| Ambiente | Databricks Free Edition (compute serverless) |
| Repositório GitHub | https://github.com/joaoaraujo21/olist-lakehouse |
| Git folder no Databricks | clone do repositório no Workspace; o desenvolvimento é feito pela interface do Databricks |
| Documentação | `docs/adr/` (decisões), `docs/design/` (desenhos técnicos por fase), `docs/dicionario_metricas.md`, `docs/progresso/` (este diário) |

## Forma de trabalho

- Cada fase começa pelo **contexto de negócio e pelo desenho técnico**, aprovado antes da implementação.
- Implementação pela interface do Databricks, com **code review** de cada entrega antes do merge.
- Toda decisão relevante é registrada em um **ADR** curto.
- **Git:** GitHub Flow — `main` + uma branch por fase/feature, PR revisado antes do merge, **Squash and merge**, apagar a branch após o merge, Conventional Commits (`feat:`, `fix:`, `docs:`...).

## Decisões em aberto (projeto)

- Janela de ação da lista de pedidos em risco (hoje a lista inclui atrasos antigos; ver dicionário v1.2) e emissão dos 52 dias restantes do stream.
- Incluir ou não a fase opcional de ML.
