# Dicionário de métricas oficiais — v1.2

- **Versão:** 1.2 (2026-11-18). A v1.2 só **acrescenta** a seção "Visão ao vivo" (monitor de pedidos em risco, Fase 5) e atualiza o período dos dados; as definições dos 8 KPIs, do scorecard, do SLA e de atraso × review são as da v1.1.
- **Versão anterior:** 1.1 (2026-11-15). A v1.1 só **acrescenta** as definições do scorecard, do SLA e de atraso × review (seção final); as definições dos 8 KPIs são as da v1.
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


## Visão ao vivo (v1.2)

Objetos da camada gold e do `ops` que sustentam o monitor de pedidos em risco (Fase 5, ADRs 0009 a 0014). São uma visão **operacional**: a fonte **oficial** dos KPIs continua sendo a gold batch (Metric Views `mv_*`, `fact_orders` e demais).

| Objeto | Grão | Para que serve |
|---|---|---|
| `olist.gold.gold_orders_live` | um pedido (só pedidos com ao menos um evento) | estado atual do pedido, mantido por streaming: status, sequência, instante de cada etapa, data estimada e nível de risco |
| `olist.gold.vw_pedidos_em_risco` | um pedido atrasado ou em risco | lista para a Logística agir antes da entrega, com os dias até a data estimada |
| `olist.ops.streaming_batches` | um micro-lote | volume, pedidos que avançaram, ignorados, eventos tardios e latência de cada ciclo |

**Qual número vale**
- Para reunião, relatório e KPI: a gold batch. A visão ao vivo **não recalcula** GMV, cancelamento, entrega no prazo ou qualquer outro KPI oficial.
- Para operar: a visão ao vivo. O status de um pedido ao vivo pode estar adiante do da gold batch até a próxima atualização dos pipelines; com a silver e a gold batch atualizadas com a mesma bronze, o status é **igual em todos os pedidos** (validado em 99.291 pedidos, etapa 5.4).
- Pedido sem nenhum evento não existe ao vivo; na gold batch ele tem status `desconhecido`.

**Regras**
- **Status atual ao vivo:** o mesmo critério dos conceitos base (evento de maior `event_sequence`). Evento duplicado, atrasado ou fora de ordem não muda o status final; o instante de cada etapa (`created_at`, `approved_at`, `shipped_at`, `delivered_at`) é preenchido por qualquer evento, na ordem em que chegar.
- **"Hoje":** a **data simulada corrente**, isto é, a data do maior `event_ts` já gravado em `gold_orders_live`. Nunca `current_date()`.
- **Pedido aberto:** status `created`, `approved`, `invoiced`, `processing` ou `shipped`. Pedido fechado (`delivered`, `canceled`, `unavailable`) tem risco nulo.
- **Nível de risco** (ADR 0011): `atrasado` se a data estimada de entrega já passou; `em_risco` se faltam de 0 a **2 dias** (N, em configuração) para a data estimada e o pedido ainda **não foi despachado** (`shipped`); `normal` nos demais pedidos abertos. Pedido despachado só vira `atrasado` depois de vencer (limitação desta versão).
- **Evento tardio:** evento com `event_ts` mais antigo que o maior `event_ts` gravado menos 4 dias. É **aplicado** e **contado** em `streaming_batches.late_events`; nunca descartado em silêncio (ADR 0013).
- **Frescor:** SLA de 15 minutos entre a gravação do arquivo na landing e a linha em `gold_orders_live` (ADR 0012); medido de 4 a 8,4 minutos por ciclo.

**Cuidado ao ler a lista**
- **"Atrasado" ao vivo não é "atraso" do SLA.** O atraso do SLA e do scorecard mede pedidos **entregues** (entrega depois da data estimada). "Atrasado" ao vivo é um pedido **ainda aberto** cuja data estimada passou; quando ele é entregue, o risco volta a nulo e o pedido entra na conta de entregues (no prazo ou não) da gold batch.
- **A lista tem pedidos antigos.** Em 2028-08-27, 1.466 dos 1.822 pedidos `atrasado` estavam atrasados havia mais de 90 dias: são pedidos do histórico que nunca foram entregues nos dados de origem. Só 104 estavam atrasados havia até 7 dias. Para acionar a Logística, use `dias_para_a_entrega` (por exemplo, `>= -7`) junto com os `em_risco`.

## O que mudou da v1.1 para a v1.2

| Tema | v1.1 | v1.2 |
|---|---|---|
| Visão ao vivo | não existia | `gold_orders_live`, `vw_pedidos_em_risco` e `ops.streaming_batches`, com as regras acima; a gold batch continua sendo a fonte oficial |
| "Atrasado" | só em pedidos entregues (`atraso_dias`) | também ao vivo, em pedidos abertos com a data estimada vencida (conceito diferente, descrito acima) |
| "Hoje" | nenhuma métrica depende de "hoje" | só a visão ao vivo depende, e é a **data simulada**, não `current_date()` |
| Período dos dados | 2026-09-06 a 2028-08-21 | 2026-09-06 a 2028-08-27, depois de emitir o stream (22 a 27/08) e atualizar a silver e a gold |
| KPIs oficiais, com os novos dias | pedidos criados 98.476; válidos 97.265; GMV R$ 15.614.632,11; entregues 93.247; no prazo 93,27% | pedidos criados 99.291; válidos 98.079; GMV R$ 15.723.094,10; entregues 94.804; no prazo 93,27% |
