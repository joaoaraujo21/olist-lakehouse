# Fase 4 — Gold e Métricas

- **Período:** 2026-11-13 a 2026-11-16
- **Status:** ✅ concluída em 2026-11-16 — [desenho técnico](../design/fase-4-gold-metricas.md) (decisões G1 a G8) implementado
- **Branch/PRs:** `fase-4/gold` (PR a abrir)

## Objetivo

**Negócio:** uma única definição para cada KPI (dicionário de métricas), disponível em tabelas prontas para análise: pedidos, itens e avaliações ligados ao cadastro **vigente na data do pedido**.

**Técnico:** modelo estrela (`fact_orders`, `fact_order_items`, `fact_reviews`, `dim_customer`, `dim_seller`, `dim_product`, `dim_date`), agregados (`gold_seller_scorecard`, `gold_delivery_sla`, `gold_daily_kpis`, `gold_atraso_vs_review`, `gold_seller_mensal`) e métricas oficiais em Metric Views, consumidas por um AI/BI Dashboard e por um espaço do Genie.

O contexto de negócio, o escopo e os critérios de aceite estão no [desenho técnico](../design/fase-4-gold-metricas.md).

## Pendências herdadas da Fase 3

Ver "Encerramento" em [fase-3-cdc-scd2.md](fase-3-cdc-scd2.md). Todas resolvidas:

- junção por intervalo (`>= __START_AT` e `< __END_AT`) para ligar pedido e item ao cadastro da data: feita uma vez na construção dos fatos, com o instante exato da mudança pertencendo à versão nova (ADR 0006);
- 288 pedidos de clientes excluídos e 280 itens de vendedores excluídos: encontram a versão fechada; a coluna `cadastro_excluido_hoje` marca o caso para a Fase 6;
- `dim_customer` e `dim_seller` partem de `silver.customers_historico` e `silver.sellers_historico`, com uma linha por versão e chave substituta.

## Etapas

| Etapa | Entrega | Status |
|---|---|---|
| 4.1 | Pipeline da gold com dimensões e fatos; verificações de granularidade e de chave | ✅ concluída em 2026-11-15 |
| 4.2 | Dicionário v1, 8 KPIs em SQL e Metric Views, validação independente | ✅ concluída em 2026-11-15 |
| 4.3 | Scorecard de vendedores, SLA de entrega e atraso × review | ✅ concluída em 2026-11-16 |
| 4.4 | Dashboard, Genie, documentação das tabelas, ADRs e encerramento | ✅ concluída em 2026-11-16 |

## O que foi feito

**Abertura (2026-11-13).** Levantamento da silver e desenho. Fatos que definiram o desenho: `silver.orders` não tem `order_status` nem datas de entrega (vêm de `order_status_events`; 2 pedidos sem evento); 745 pedidos sem item; 17 reviews sem pedido; 544 pedidos com mais de uma review; 623 produtos sem tradução de categoria (610 sem categoria e 13 de 2 categorias não traduzidas); GMV bruto de R$ 15,72 milhões em 97.731 pedidos com item e 3.077 vendedores. Levantamento adicional na 4.1: os 2 pedidos sem evento têm 1 item cada; o par (`review_id`, `order_id`) é único (95.253), embora o `review_id` se repita; 1.370 pedidos têm inversões entre `event_ts` e `event_sequence` (127 no evento final), e só 223 têm falha registrada pelo simulador, a maior parte vem do dataset original.

**4.1 — modelo dimensional.** Pipeline `olist.gold` (fonte `src/gold/01_gold_modelo`) com 7 materialized views: `dim_date` (812 dias), `dim_product` (32.951), `dim_customer` (101.008 versões), `dim_seller` (3.361), `fact_orders` (98.476), `fact_order_items` (111.592) e `fact_reviews` (95.253). Status atual do pedido = evento de maior `event_sequence`; data de entrega = `event_ts` do evento `delivered`. Fatos sem dimensão: 0; chaves substitutas únicas; GMV bruto R$ 15.722.601,92 nos itens e nos pedidos. Segunda execução sem dado novo: as 7 views em `NO_OP`, mesmas contagens.

**4.2 — KPIs e validação independente.** Dicionário de métricas v1 e ADR 0007. `fact_orders` ganhou `pedidos_validos_do_cliente` (base da recompra) e nasceu `gold_seller_mensal` (base da concentração). Notebook `02_metric_views` cria `mv_pedidos`, `mv_itens` e `mv_concentracao` (formato YAML 1.1). Valores oficiais: GMV R$ 15.614.632,11 (igual em `mv_pedidos` e `mv_itens`), 97.265 pedidos válidos, ticket R$ 160,54, cancelamento 0,59%, 93,27% entregues no prazo, lead time 12,67 dias (média) e 10,31 (mediana), review 4,15, recompra 3,04% (2.857 de 94.084 clientes), concentração do top 10 mensal 18,14%. `quality/04_validacao_gold` recalcula tudo direto da silver (por caminhos diferentes dos da gold) e grava em `quality.validacao_gold`: 50 verificações OK, 0 reprovadas.

**4.3 — scorecard, SLA e atraso × review.** Fonte `src/gold/03_gold_agregados` com `gold_seller_scorecard` (3.127 versões de 3.077 vendedores), `gold_delivery_sla` (553 grupos mês × UF), `gold_daily_kpis` (608 dias, colunas aditivas) e `gold_atraso_vs_review` (6 faixas). Decisão da opção A para pedidos compartilhados (ADR 0008, dicionário v1.1). Critério 6: pelo instante da compra, 4.022 itens antes e 7.231 depois da primeira mudança de endereço, nos 265 vendedores que mudaram e venderam. Resultado de negócio: a nota média da review cai de 4,29 (pedido adiantado) para 4,00 (no dia), 3,26 (1 a 3 dias de atraso), 2,11 (4 a 7), 1,65 (8 a 14) e 1,71 (15 ou mais). Validação com 111 linhas, 0 reprovadas, 13 verificações de idempotência iguais.

**4.4 — consumo, documentação e encerramento.**
- **AI/BI Dashboard** "Olist Lakehouse - Métricas Oficiais" (3 páginas: visão executiva com os 8 KPIs, entrega e atraso, scorecard de vendedores; 19 widgets; 9 datasets que consultam as Metric Views com `MEASURE()` e os agregados), criado como rascunho por API.
- **Espaço do Genie** com o mesmo nome: 7 tabelas (3 Metric Views e 4 agregados, **sem** as dimensões com dado pessoal), instruções com as definições do dicionário v1.1 e 8 consultas de exemplo. A pergunta "Qual o GMV total e a taxa de recompra?" devolveu R$ 15.614.632,11 e 3,04%, os números das Metric Views.
- **Documentação:** notebook `05_documentacao_gold` comenta as 145 colunas das 12 tabelas e aplica as tags `camada`, `tipo_tabela`, `contem_pii`, `pii` e `pii_categoria` (11 colunas de PII em 4 tabelas: `dim_customer` 4, `dim_seller` 3, `gold_seller_scorecard` 3, `fact_orders` 1).
- **Validação:** o grupo 9 do `04_validacao_gold` lê o dashboard, executa os 9 datasets e compara os 8 KPIs com os das Metric Views (critério 5 do desenho). Resultado no workspace: 121 linhas, 102 verificações OK, 0 reprovadas, 19 fotos; a segunda execução inclui 13 de idempotência e as 10 do dashboard.
- ***Full refresh*:** duas execuções (as 12 tabelas recalculadas do zero, cerca de 1 minuto cada). Depois da segunda, antes de reaplicar qualquer coisa: 145 de 145 colunas comentadas, 36 tags de tabela e 22 de coluna intactas, dados idênticos e a validação inteira (108 linhas) sem reprovação.
- **Versionamento:** as definições do dashboard (`.lvdash.json`) e do Genie (`.geniespace.json`) foram exportadas do workspace e entram no repositório em `resources/` (`olist_metricas.lvdash.json` e `olist_metricas.geniespace.json`), prontas para o bundle da Fase 7.

## Decisões

| # | Decisão | Onde |
|---|---|---|
| G1 | Materialized views em pipeline próprio `olist.gold` (o Free Edition aceitou o segundo pipeline; o plano reserva não foi usado) | desenho |
| G2 | Status atual = maior `event_sequence`; data de entrega = `event_ts` do evento `delivered` | desenho, ADR 0007 |
| G3 | Fatos de item, pedido e review; dimensões de data, produto, cliente e vendedor (as duas últimas por versão) | desenho |
| G4 | Chave substituta por versão e junção por intervalo uma vez na construção | desenho, ADR 0006 |
| G5 | Dicionário v1 fecha as pontas em aberto; pedido sem evento é válido; concentração por mês; recompra em qualquer período | ADR 0007, dicionário v1 |
| G6 | 8 KPIs em Metric Views; dashboard e Genie consomem as Metric Views | desenho |
| G7 | Validação independente em `quality.validacao_gold`, incluindo o dashboard | desenho |
| G8 | `CLUSTER BY` nos fatos e agregados (ganho educativo; ajuste fino na Fase 8) | desenho |
| — | Pedido com itens de mais de um vendedor entra no scorecard de cada vendedor (opção A); cadastro do vendedor pela data da compra | ADR 0008, dicionário v1.1 |
| — | O texto do comentário da review não vai para a gold (só `tem_comentario`); o Genie não recebe as dimensões com dado pessoal | 4.1, 4.4 |
| — | Comentário de tabela é do pipeline (`comment=`); comentário de coluna e tags são aplicados por notebook | 4.4 |

## Problemas encontrados e como foram resolvidos

- **Primeira execução do pipeline falhou** na inicialização do Unity Catalog (`PERMISSION_DENIED: Can not move tables across arclight catalogs`), antes de rodar qualquer código; o *Dry run* tinha passado. Apagar o pipeline e recriá-lo com catálogo `olist` e schema `gold` definidos na criação, sem editar as configurações depois, resolveu. Hipótese: trocar catálogo ou schema depois do primeiro *Dry run* dispara a movimentação de tabelas, que o Free Edition não permite.
- **YAML das Metric Views:** o formato `0.1` não aceita `comment` nas colunas (o `1.1` aceita), e um `comment` com dois-pontos no meio do texto é lido como um novo campo (`mapping values are not allowed here`). Textos reescritos e aviso no notebook.
- **`SHOW VIEWS IN olist.gold` não aceita catálogo no nome:** resolvido com `USE CATALOG` e `SHOW VIEWS IN gold`.
- **Validação falhou na 4.3** porque o pipeline tinha rodado só em *Dry run*: as tabelas novas não existiam. Um *Dry run* só tem eventos de definição; a execução real tem `SETTING_UP_TABLES`, `RUNNING` e `flow_progress`.
- **Critério 6 não se reproduzia** com os números da Fase 3 (3.906 e 7.347): ela usou o limite de envio como data da venda; a gold usa o instante da compra (4.022 e 7.231). O total, 11.253 itens, é o mesmo. Critério reescrito (ADR 0008).
- **Texto impreciso:** "mudaram de cidade". Dos 266 vendedores com mais de uma versão, 251 mudaram de cidade (165 de UF) e 15 só de CEP; o texto passou a dizer "mudaram de endereço".
- **Contagem de colunas:** as 12 tabelas têm 145 colunas, não 150, como apareceu em alguns registros.
- **Última célula do `05_documentacao_gold` falhava sozinha** (`NameError` numa sessão nova, porque dependia de variáveis de outras células): passou a ser autossuficiente e a listar as tabelas direto do catálogo.
- **Dashboard com duas tabelas sem exibir dados:** a interface reescreveu as tabelas de atraso × review e do scorecard para um formato novo, sem colunas e sem consulta. Causa provável do problema original: colunas `double` sem `numberFormat`. As tabelas foram reconstruídas com `numberFormat`, mas o problema persistiu na tela. Limitação conhecida e aceita nesta fase: os números do dashboard são validados pelos datasets (grupo 9), e o consumo principal por BI externo é o próximo passo.

## Aprendizados

- **Os dados mandam no desenho:** o status e a data de entrega não existiam como colunas; reconstruí-los dos eventos exigiu um ADR e uma versão nova do dicionário antes de qualquer KPI.
- **Uma definição, vários consumidores:** SQL, Metric View, dashboard e Genie dão o mesmo número porque todos leem a mesma definição, e uma validação por outro caminho prova isso (não repete o código da gold).
- **Agregações sobre agregações** (concentração) e **contagens por cliente em qualquer período** (recompra) não cabem numa medida simples: pedem uma tabela de apoio ou uma coluna calculada na fato.
- **Colunas aditivas** (contagens e somas) permitem recompor qualquer razão em qualquer período; médias não se somam.
- **`NO_OP` não é prova** de que o resultado seria igual se recalculado; o *Full refresh* com a validação por cima é.
- **O que sobrevive a um *Full refresh*** numa materialized view: comentários e tags de coluna e tags de tabela sim; o comentário de tabela é regravado a partir do pipeline.
- **Testar sem a tela:** o que só se vê na interface (dashboard, Genie) precisa de um teste de dados por trás (datasets executados e comparados) e de uma conferência visual separada.

## Encerramento

Todos os critérios de aceite do desenho foram atendidos (fact_orders 98.476, fact_order_items 111.592, 0 fatos sem dimensão, GMV igual, 8 KPIs idênticos em SQL, Metric View e dashboard, critério 6 reescrito pelo instante da compra, execução repetida sem mudança).

Pendências para as próximas fases:

- **Fase 5 (streaming):** `gold_orders_live` e o monitor de pedidos em risco; os últimos 2 meses do dataset; a taxa de cancelamento e o atraso mudam conforme chegam eventos (o GMV de um mês pode diminuir quando um pedido válido é cancelado).
- **Fase 6 (governança e LGPD):** `dim_customer`, `dim_seller`, `fact_orders.customer_unique_id` e o scorecard têm PII (`localizacao` e `identificador`) e guardam endereços antigos; falta máscara por categoria, perfis de acesso, exclusão ponta a ponta (inclusive nas dimensões por versão e no `VACUUM`) e a decisão sobre pedidos de cliente excluído (`cadastro_excluido_hoje`); decidir o que o Genie e os BI externos podem ler; as tabelas internas do pipeline (`__materialization_*`, `event_log_*`) ficam fora dos grants.
- **Fase 7 (testes, CI/CD, orquestração):** testes unitários das funções de transformação; módulo único (hoje a configuração e as funções se repetem entre notebooks); job que orquestra pipeline, documentação e validação (as duas validações seguidas provam a idempotência); bundle com `resources.dashboards` e `resources.genie_spaces` a partir de `resources/`; proteção da `main`.
- **Fase 8 (performance):** medir o `CLUSTER BY` das fatos e agregados (hoje só educativo, com cerca de 100 mil linhas) e o custo de recomputar os agregados.
- **Dashboard:** corrigir as duas tabelas que não exibem dados (testar o formato novo de tabela da interface) e publicar; o consumo por Power BI é o próximo passo de teste.
