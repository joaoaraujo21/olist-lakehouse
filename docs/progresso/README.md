# Progresso do projeto — Olist Lakehouse

Índice do diário de bordo. Cada fase tem seu próprio arquivo com o que foi feito, as decisões e as pendências; este arquivo guarda só o que vale para o projeto todo.

**Última atualização:** 2026-11-02 · **Fase atual:** [Fase 1 — Simulador e Bronze](fase-1-simulador-bronze.md) (não iniciada)

## Fases

| Fase | Status | Registro |
|---|---|---|
| 0 — Fundação | ✅ concluída em 2026-11-02 | [fase-0-fundacao.md](fase-0-fundacao.md) |
| 1 — Simulador e Bronze | ⏳ próxima | [fase-1-simulador-bronze.md](fase-1-simulador-bronze.md) |
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
| Workspace Databricks (Free Edition) | `https://dbc-320fbd50-0c3f.cloud.databricks.com` — usuário `joaogui21@hotmail.com` |
| Repositório GitHub (público) | https://github.com/joaoaraujo21/olist-lakehouse |
| Git folder no Databricks | clone do repositório acima no Workspace do usuário (o trabalho é feito pela interface) |
| Pasta local | `D:\Projeto Databricks - Olist` — planejamento (`.docx`), rascunhos de código/docs, `progresso/` e `CLAUDE.md` |
| CLI local | Databricks CLI v1.14.0, perfil `joaogui21@hotmail.com` (login OAuth; se expirar: `databricks auth login --host <url> --profile joaogui21@hotmail.com`). `gh` não está instalado; o GitHub é lido pela API pública (`curl`) |

## Como trabalhamos (combinado)

- Tudo em **português**.
- João implementa pela **interface do Databricks** e prefere **digitar o código ele mesmo**; Claude apresenta o desafio, dá dicas e referências, faz **code review como tech lead** (workspace pela CLI, GitHub pela API) e ajuda nos ADRs. Código/documento completo só quando pedido.
- Sem pedido explícito para executar, Claude só **lê/analisa** o workspace.
- Explicações passo a passo para operações na interface (Databricks e GitHub).
- **Git:** GitHub Flow — `main` + uma branch por fase/feature, PR revisado antes do merge, **Squash and merge**, apagar a branch após o merge, Conventional Commits (`feat:`, `fix:`, `docs:`...).

## Decisões em aberto (projeto)

- SLA do monitor quase em tempo real (5 vs 15 min) — decidir até a Fase 5.
- Incluir ou não a fase opcional de ML.
