# Progresso do projeto — Olist Lakehouse

Índice do diário de bordo. Cada fase tem seu próprio arquivo com o que foi feito, as decisões e as pendências; este arquivo guarda só o que vale para o projeto todo.

**Última atualização:** 2026-11-08 · **Fase atual:** [Fase 1 — Simulador e Bronze](fase-1-simulador-bronze.md) (em andamento — etapa 1.4 concluída)

## Fases

| Fase | Status | Registro |
|---|---|---|
| 0 — Fundação | ✅ concluída em 2026-11-02 | [fase-0-fundacao.md](fase-0-fundacao.md) |
| 1 — Simulador e Bronze | 🔄 em andamento | [fase-1-simulador-bronze.md](fase-1-simulador-bronze.md) |
| 2 — Silver e Qualidade | — | |
| 3 — CDC e SCD2 | — | |
| 4 — Gold e Métricas | — | |
| 5 — Streaming | — | |
| 6 — Governança e LGPD | — | |
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

- SLA do monitor quase em tempo real (5 vs 15 min) — decidir até a Fase 5.
- Incluir ou não a fase opcional de ML.
