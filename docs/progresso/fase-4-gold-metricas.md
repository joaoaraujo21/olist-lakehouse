# Fase 4 — Gold e Métricas

- **Período previsto:** a definir na abertura da fase
- **Status:** a abrir
- **Branch/PRs:** —

## Objetivo

**Negócio:** uma única definição para cada KPI (dicionário de métricas), disponível em tabelas prontas para análise: pedidos, itens e avaliações ligados ao cadastro **vigente na data do pedido**.

**Técnico:** modelo estrela (`fact_orders`, `fact_order_items`, `fact_reviews`, `dim_customer`, `dim_seller`, `dim_product`, `dim_date`), agregados (`gold_seller_scorecard`, `gold_delivery_sla`, `gold_daily_kpis`) e métricas oficiais em Metric Views.

O contexto de negócio, o escopo e os critérios de aceite serão detalhados na abertura da fase.

## Pendências herdadas da Fase 3

Ver "Encerramento" em [fase-3-cdc-scd2.md](fase-3-cdc-scd2.md), em especial:

- junção por intervalo (`>= __START_AT` e `< __END_AT`) para ligar pedido e item ao cadastro da data, com a convenção da fronteira documentada;
- 288 pedidos de clientes excluídos e 280 itens de vendedores excluídos: a versão existe, mas está fechada;
- `dim_customer` e `dim_seller` partem de `silver.customers_historico` e `silver.sellers_historico`.

## Etapas

_(definir na abertura da fase)_

## O que foi feito

_(preencher durante a fase)_

## Decisões

_(preencher durante a fase)_

## Problemas encontrados e como foram resolvidos

_(preencher durante a fase)_

## Aprendizados

_(preencher durante a fase)_

## Encerramento

_(preencher ao concluir)_
