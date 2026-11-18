# Olist Lakehouse

Plataforma Lakehouse no **Databricks** construída sobre o dataset público de e-commerce da [Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) (~100 mil pedidos, 2016–2018). Projeto de estudo e portfólio em engenharia de dados.

## O problema de negócio

Um marketplace brasileiro em que Comercial, Logística e Financeiro calculam os mesmos KPIs de formas diferentes, atrasos de entrega são descobertos tarde, não há scorecard confiável de vendedores e dados pessoais circulam sem controle (LGPD). O objetivo é uma plataforma única, governada e reproduzível por código, com métricas oficiais de definição única.

Como o dataset é histórico, um **simulador** reencena os pedidos "no presente", gerando arquivos aos poucos — com duplicatas, atrasos e eventos fora de ordem — para reproduzir os desafios de um ambiente de produção.

## Arquitetura

```
Simulador Python ──► Volume landing ──► BRONZE ──► SILVER ──► GOLD ──► Dashboard / Genie / SQL
                        Auto Loader      Lakeflow Declarative     Star schema +
                                         Pipelines + expectations  Metric Views
Transversal: Unity Catalog (governança/LGPD) · Lakeflow Jobs · Asset Bundles + GitHub Actions
```

Unity Catalog: catálogo `olist` com os schemas `landing`, `bronze`, `silver`, `gold`, `quality` (qualidade e quarentena) e `ops` (logs de execução).

## Estrutura do repositório

| Pasta | Conteúdo |
|---|---|
| `setup/` | Criação da estrutura e carga do dataset original (`00_setup_olist`); reset da simulação (`99_reset_simulacao`) |
| `src/simulador/` | Simulador que reencena os pedidos e injeta falhas controladas, com gabarito em `ops.simulator_chaos_log`; modo `batch` (dias inteiros) e modo `stream` (fatias de horas) |
| `src/bronze/` | Ingestão incremental com Auto Loader (10 tabelas, bruto + metadados); caminho rápido com só as entidades do monitor |
| `src/silver/` | Pipeline declarativo da silver: tipagem, deduplicação, AUTO CDC (SCD1) e quarentena, e histórico de cadastro (SCD2) |
| `src/gold/` | Modelo dimensional (materialized views), agregados e Metric Views dos KPIs oficiais |
| `src/streaming/` | Monitor ao vivo: `gold_orders_live` (Structured Streaming, `foreachBatch` + `MERGE` idempotente) e a lista de pedidos em risco de atraso |
| `src/quality/` | Resultados de qualidade, validação independente da silver, da gold e do monitor ao vivo, e documentação (comentários e tags) |
| `resources/` | Definições versionadas do AI/BI Dashboard (`.lvdash.json`) e do espaço do Genie (`.geniespace.json`) |
| `docs/adr/` | Registros de decisões de arquitetura (ADRs) |
| `docs/design/` | Desenho técnico de cada fase |
| `docs/progresso/` | Diário de bordo: o que foi feito, decisões e pendências por fase |
| `docs/dicionario_metricas.md` | Definição oficial dos KPIs e da visão ao vivo (v1.2) |

## Como executar

Ambiente: Databricks Free Edition (compute serverless).

1. Clone este repositório como **Git folder** no workspace.
2. Abra `setup/00_setup_olist` e confira no painel **Environment** a dependência `kagglehub==1.0.3`. Execute todas as células (idempotente; valida a contagem de linhas dos 9 arquivos).
3. Execute o simulador (`src/simulador/01_simulador_base`) para gerar os arquivos na landing e, em seguida, a ingestão bronze (`src/bronze/01_bronze_ingestao`).
4. Crie o pipeline declarativo `olist.silver` com `src/silver/01_silver_pipeline` e `src/silver/02_silver_historico` como fontes e execute-o.
5. Rode os notebooks de `src/quality/` (resultados de qualidade, validação da silver, documentação de PII).
6. Crie o pipeline declarativo `olist.gold` com `src/gold/01_gold_modelo` e `src/gold/03_gold_agregados` como fontes e execute-o; depois rode `src/gold/02_metric_views` (Metric Views dos KPIs), `src/quality/04_validacao_gold` (validação independente) e `src/quality/05_documentacao_gold` (comentários e tags).
7. O dashboard e o espaço do Genie podem ser recriados a partir de `resources/`.

8. Streaming (Fase 5): execute o simulador com `modo = stream` (uma fatia de horas por vez, ou várias com `fatias_por_execucao`), a bronze com `entidades = order_status_events,orders` e `src/streaming/01_gold_orders_live` (`teste_leitura = nao`) para atualizar `gold_orders_live` e a view `vw_pedidos_em_risco`; `src/quality/06_validacao_live` valida o monitor e `07_documentacao_live` aplica comentários e tags. Os testes do monitor rodam no próprio notebook (`teste_leitura`).

O passo a passo de cada fase e as decisões estão em `docs/progresso/` e `docs/adr/`.

## Roadmap

- [x] Fase 0 — Fundação: catálogo, schemas, landing, métricas oficiais v0
- [x] Fase 1 — Simulador e ingestão bronze (Auto Loader)
- [x] Fase 2 — Silver e qualidade de dados
- [x] Fase 3 — CDC e SCD2
- [x] Fase 4 — Gold, métricas e dashboard
- [x] Fase 5 — Streaming e monitor de atrasos
- [ ] Fase 6 — Governança e LGPD
- [ ] Fase 7 — Testes, CI/CD e orquestração
- [ ] Fase 8 — Performance

## Dados e licença

Dataset [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), licença CC BY-NC-SA 4.0. Uso exclusivamente educacional, com atribuição à Olist.
