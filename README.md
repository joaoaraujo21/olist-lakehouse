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
| `setup/` | Notebook de criação da estrutura e carga do dataset original |
| `docs/adr/` | Registros de decisões de arquitetura (ADRs) |
| `docs/dicionario_metricas.md` | Definição oficial dos KPIs |

## Como executar

Ambiente: Databricks Free Edition (compute serverless).

1. Clone este repositório como **Git folder** no workspace.
2. Abra `setup/00_setup_olist` e confira no painel **Environment** a dependência `kagglehub==1.0.3`.
3. Execute todas as células. O notebook é idempotente e termina validando a contagem de linhas dos 9 arquivos.

## Roadmap

- [x] Fase 0 — Fundação: catálogo, schemas, landing, métricas oficiais v0
- [x] Fase 1 — Simulador e ingestão bronze (Auto Loader)
- [ ] Fase 2 — Silver e qualidade de dados
- [ ] Fase 3 — CDC e SCD2
- [ ] Fase 4 — Gold, métricas e dashboard
- [ ] Fase 5 — Streaming e monitor de atrasos
- [ ] Fase 6 — Governança e LGPD
- [ ] Fase 7 — Testes, CI/CD e orquestração
- [ ] Fase 8 — Performance

## Dados e licença

Dataset [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), licença CC BY-NC-SA 4.0. Uso exclusivamente educacional, com atribuição à Olist.
