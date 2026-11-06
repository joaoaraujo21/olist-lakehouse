# Fase 0 — Fundação

- **Período:** 2026-11-02 (1 sessão)
- **Status:** ✅ concluída
- **Branch/PRs:** `fase-0/setup` — PR #1 e PR #2 (mergeados)

## Objetivo

**Negócio:** entender o marketplace e fixar as métricas oficiais.
**Técnico:** catálogo `olist` com schemas, Volume de landing, repositório GitHub.
**Entrega prevista:** estrutura do repo + README inicial + dicionário de métricas v0.

## O que foi feito

### Databricks
- **Catálogo `olist`** com schemas `landing`, `bronze`, `silver`, `gold`, `quality`, `ops`, todos com `COMMENT`. Schema `default` removido.
- **Volumes:**
  - `olist.landing.olist_source` — os 9 CSVs originais (Kaggle `olistbr/brazilian-ecommerce`, versão 2). Somente leitura; fonte do simulador.
  - `olist.landing.raw_data` — vazio; zona de chegada do simulador (uma subpasta por entidade), lida pelo Auto Loader.
- **Notebook `setup/00_setup_olist`** (Git folder): cria a estrutura com `IF NOT EXISTS`, documenta com `COMMENT ON`, baixa o dataset com cache em `/tmp` (apagado ao final, via `tempfile.mkdtemp()`), confere os 9 arquivos esperados e valida contagens de linhas. **Idempotente.** Dependência `kagglehub==1.0.3` no painel Environment (serverless versão 6).
- **Validação de contagem 9/9 OK:** orders 99.441 · customers 99.441 · order_items 112.650 · payments 103.886 · reviews 99.224 · products 32.951 · sellers 3.095 · geolocation 1.000.163 · translation 71. (Leitura com `multiLine` + `escape='"'` — as reviews têm quebras de linha no texto.)

### Repositório (https://github.com/joaoaraujo21/olist-lakehouse)
```
README.md                                          problema de negócio, arquitetura, execução, roadmap, licença
setup/00_setup_olist.ipynb                         notebook de setup (.ipynb guarda os metadados do Environment)
docs/adr/0001-organizacao-landing.md               organização da landing
docs/adr/0002-pedido-valido-e-entrega-no-prazo.md  definições de negócio + regras de qualidade
docs/dicionario_metricas.md                        dicionário v0 dos 8 KPIs
```
- **PR #1** — setup + ADRs + dicionário (mergeado antes da revisão).
- **PR #2** — README + correções do notebook (revisado, aprovado; mergeado como merge commit).

## Decisões

| Decisão | Registro |
|---|---|
| Landing com 2 volumes: `olist_source` (estático) e `raw_data` (simulador, subpasta por entidade) | ADR 0001 |
| Versão do dataset fixada (`versions/2`); cache do kagglehub em disco local temporário | ADR 0001 |
| **Pedido válido** = status ≠ `canceled`/`unavailable` **e** ≥ 1 item | ADR 0002 |
| **Pedido entregue** = `delivered` **e** data de entrega ao cliente não nula | ADR 0002 |
| **Entregue no prazo** = `DATE(entrega) <= DATE(estimada)` (comparação por dia) | ADR 0002 |
| GMV = `price + freight_value` (não `payment_value`); recompra por `customer_unique_id` | Dicionário v0 |
| Dependências no painel Environment (não `%pip`), com versão fixada | sessão |
| Fluxo Git: GitHub Flow + Squash and merge + Conventional Commits | sessão |

## Problemas encontrados e como foram resolvidos

| Problema | Solução |
|---|---|
| `CREATE` sem `IF NOT EXISTS` — notebook falhava na 2ª execução | `IF NOT EXISTS` em todos os `CREATE`; célula `DROP SCHEMA default` removida (uso único) |
| CSVs soltos na raiz de `raw_data` + `.kagglehub_cache` com cópia duplicada | Volume `olist_source` separado; cache em `/tmp`; `raw_data` limpo |
| Download sempre da "última versão" | Versão fixada `versions/2` |
| `%pip install` comentado (não fazia nada) e lib sem versão | Painel Environment com `kagglehub==1.0.3`; versão citada no cabeçalho do notebook |
| Linhas `# DBTITLE` / `# MAGIC` digitadas como texto | Explicado o formato source `.py`; linhas removidas |

## Aprendizados

- Formato source do Databricks (`# Databricks notebook source`, `# COMMAND ----------`, `# MAGIC`, `# DBTITLE`) e magic commands (`%sql`, `%md`, `%pip`).
- Painel **Environment** do serverless: versão do ambiente + dependências salvas no notebook.
- Git folder: branch, commit & push, pull pela interface; ciclo de vida de PR (manter aberto, novos commits na mesma branch, re-request review), tipos de merge.

## Encerramento — pendências herdadas para a Fase 1

- [ ] Apagar a branch `fase-0/setup` no GitHub.
- [ ] Configurar proteção da `main` (Settings → Branches: exigir PR antes do merge).
- [ ] Confirmar que o notebook antigo `Projeto Olist/Estrutura Olist & Landing Zone` (fora do Git folder) foi apagado.
- [ ] No Databricks: Git folder → `main` → **Pull**.
- [ ] Opcionais no notebook de setup (no próximo PR que mexer nele): crases em `` `kagglehub==1.0.3` ``, linha em branco no topo da célula de validação, títulos nas células.
