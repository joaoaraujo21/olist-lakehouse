# Dicionário de métricas oficiais — v1.1

- **Versão:** 1.1 (2026-11-15). A v1.1 só **acrescenta** as definições do scorecard, do SLA e de atraso × review (seção final); as definições dos 8 KPIs são as da v1.
- **Conceitos base:** ver [ADR 0002](adr/0002-pedido-valido-e-entrega-no-prazo.md), [ADR 0007](adr/0007-derivacao-de-status-e-definicoes-de-borda-da-gold.md) e, para o scorecard, [ADR 0008](adr/0008-scorecard-de-vendedores-pedidos-compartilhados-e-cadastro-na-compra.md)
- **Onde cada métrica é calculada:** Metric Views `olist.gold.mv_pedidos`, `olist.gold.mv_itens` e `olist.gold.mv_concentracao`, sobre as tabelas da camada gold. Dashboard, Genie e consultas SQL usam essas definições, e a validação (`quality.validacao_gold`) recalcula os números por outro caminho.

Toda mudança de definição gera uma nova versão deste arquivo e, se for uma decisão relevante, um novo ADR.

## O que mudou da v0 para a v1

| Tema | v0 | v1 |
|---|---|---|
| Status e data de entrega | supunha as colunas `order_status` e `order_delivered_customer_date` | a silver guarda eventos; o status atual é o do evento de **maior `event_sequence`** e a data de entrega é o `event_ts` do evento `delivered` |
| Pedido sem nenhum evento | não tratado | status `desconhecido`; conta como pedido criado e, se tiver item, é **válido** (a regra do ADR 0002 só exclui `canceled` e `unavailable`) |
| Pedido sem item | "fora de pedido válido" | continua fora de pedido válido e **conta no total de pedidos criados** da taxa de cancelamento |
| Mais de uma review por pedido | vale a mais recente por `review_answer_timestamp` | mesmo critério, com desempate por `review_creation_date` e depois `review_id` |
| Review sem pedido | não tratado | fora de todas as métricas (`pedido_existe = false`) |
| Fuso horário | `America/Sao_Paulo` | os timestamps da silver já estão no relógio local do Brasil: **não há conversão** |
| Concentração de vendas | "calculada por período" | definida **mês a mês** (top 10 do mês); sem recorte, soma os meses |
| Recompra e cliente excluído | não tratado | o cliente excluído continua contando no passado, pelo `customer_unique_id` da versão do cadastro vigente na compra |
| Datas | — | nenhuma métrica depende de "hoje"; o período é a data da compra e os dados vão de 2026-09-06 a 2028-08-21 |

## Conceitos base

| Conceito | Definição |
|---|---|
| **Status atual** | `status` do evento de maior `event_sequence` do pedido; `desconhecido` se o pedido não tem evento |
| **Pedido válido** | tem ao menos 1 item **e** o status atual não é `canceled` nem `unavailable` |
| **Pedido entregue** | status atual `delivered` **e** data de entrega (`event_ts` do evento `delivered`) preenchida |
| **Entregue no prazo** | pedido entregue com `DATE(entrega) <= DATE(entrega estimada)` |
| **Período de referência** | data da compra (`order_purchase_timestamp`), sem conversão de fuso |
| **Cliente único** | `customer_unique_id` da versão do cadastro vigente na compra |

## Métricas

| Métrica | Fórmula | Universo | Medida na Metric View |
|---|---|---|---|
| **GMV** | Σ (`price` + `freight_value`) dos itens | pedidos válidos | `gmv` (`mv_pedidos`, `mv_itens`) |
| **Ticket médio** | GMV ÷ nº de pedidos válidos | pedidos válidos | `ticket_medio` |
| **Taxa de cancelamento** | pedidos `canceled` ÷ todos os pedidos criados | todos os pedidos | `taxa_cancelamento` (`unavailable` não entra) |
| **% entregas no prazo** | entregues no prazo ÷ pedidos entregues | pedidos entregues | `pct_entregas_no_prazo` |
| **Lead time de entrega** | dias (com fração) entre a compra e a entrega | pedidos entregues | `lead_time_medio_dias` e `lead_time_mediana_dias` |
| **Review score médio** | média de `review_score` (1 a 5) da review mais recente do pedido | pedidos entregues com review | `review_score_medio` |
| **Taxa de recompra** | clientes únicos com 2+ pedidos válidos ÷ clientes únicos com 1+ pedido válido | pedidos válidos | `taxa_recompra` |
| **Concentração de vendas** | GMV dos 10 vendedores de maior GMV do mês ÷ GMV do mês | pedidos válidos | `concentracao_top10` (`mv_concentracao`) |

Observações:
- **Taxa de recompra** conta o cliente que tem 2 ou mais pedidos válidos **em qualquer período**. Num recorte por mês, aparecem os clientes que compraram naquele mês e que, no total, compraram 2 ou mais vezes.
- **Concentração** sem recorte soma, mês a mês, o GMV do top 10 do mês e divide pelo GMV total. O desempate entre vendedores com o mesmo GMV no mês é por `seller_id`.
- **Ticket médio por vendedor ou categoria** (`mv_itens`) divide o GMV do recorte pelos pedidos válidos que têm item no recorte; um pedido com dois vendedores conta nos dois.
- O GMV de um mês pode **diminuir** depois: um pedido válido que vira `canceled` sai do GMV do mês da compra (ADR 0002).

## Recortes padrão

Todas as métricas podem ser recortadas por período (dia, mês e ano da compra). Os demais recortes dependem do grão da Metric View:

| Recorte | Disponível em |
|---|---|
| UF do cliente na data da compra, status atual | `mv_pedidos` (todos os KPIs de pedido) |
| Categoria do produto (inglês), vendedor, UF do vendedor na data da compra | `mv_itens` (GMV, itens, pedidos válidos e ticket médio) |
| Mês | `mv_concentracao` |

Entrega, cancelamento e review **por vendedor** ficam no scorecard (etapa 4.3), porque um pedido pode ter itens de vários vendedores.

## Scorecard, SLA e atraso × review (v1.1)

Tabelas da camada gold que respondem às perguntas de Comercial, Logística e Atendimento. Usam as mesmas definições de pedido válido, entregue e no prazo dos 8 KPIs.

| Tabela | Grão | Para que serve |
|---|---|---|
| `gold_seller_scorecard` | uma linha por **versão do cadastro** do vendedor que vendeu | GMV, cancelamento, entrega no prazo, atraso e review por vendedor, com a cidade e a UF da data da venda |
| `gold_delivery_sla` | mês da compra × UF do cliente na compra | entregues, no prazo, atrasados, lead time e atraso |
| `gold_daily_kpis` | dia da compra | contagens e somas aditivas; qualquer razão se recompõe somando e dividindo depois |
| `gold_atraso_vs_review` | faixa de atraso | nota média da review por faixa de atraso |

Regras do scorecard (ADR 0008):
- **Pedido compartilhado:** o pedido entra no scorecard de **cada** vendedor que tem item nele, com o status, o atraso e a review do pedido inteiro. Por isso a soma de `pedidos` dos vendedores (99.049) é maior que o total de pedidos com item (97.731); `pedidos_compartilhados` conta, em cada vendedor, os pedidos que têm mais de um vendedor (1.256 pedidos, que dão 2.574 pares pedido-vendedor).
- **GMV:** soma apenas os itens do próprio vendedor, em pedidos válidos; a soma do scorecard é igual ao GMV oficial.
- **Cadastro na data da venda:** a venda vai para a versão do vendedor vigente no **instante da compra**; o instante exato da mudança pertence à versão nova.
- `taxa_cancelamento` = pedidos `canceled` do vendedor ÷ pedidos do vendedor; `pct_entregas_no_prazo` = entregues no prazo ÷ entregues; `review_score_medio` = média da review mais recente dos pedidos entregues do vendedor.

Regras de atraso:
- **Atraso** = `DATE(entrega) − DATE(entrega estimada)` em dias, só para pedidos entregues. Positivo = atrasou; zero = no dia; negativo = adiantado.
- **Faixas** de `gold_atraso_vs_review`: `1_adiantado` (−1 ou menos), `2_no_dia` (0), `3_atraso_1_a_3_dias`, `4_atraso_4_a_7_dias`, `5_atraso_8_a_14_dias`, `6_atraso_15_dias_ou_mais`. Só entram pedidos entregues **com** review.
- Em `gold_delivery_sla`, `atraso_medio_dos_atrasados_dias` mede só quem atrasou; a média de todos misturaria atraso com antecipação.
