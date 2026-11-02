# Dicionário de métricas oficiais — v0

- **Versão:** 0 (2026-11-02)
- **Conceitos base:** ver [ADR 0002](adr/0002-pedido-valido-e-entrega-no-prazo.md)

Toda mudança de definição gera uma nova versão deste arquivo e, se for uma decisão relevante, um novo ADR.

## Conceitos base

| Conceito | Definição |
|---|---|
| **Pedido válido** | `order_status` não é `canceled` nem `unavailable` **e** o pedido tem ao menos 1 item |
| **Pedido entregue** | `order_status = 'delivered'` **e** `order_delivered_customer_date` não nulo |
| **Entregue no prazo** | pedido entregue com `DATE(order_delivered_customer_date) <= DATE(order_estimated_delivery_date)` |
| **Período de referência** | data de compra (`order_purchase_timestamp`), em `America/Sao_Paulo` |
| **Cliente único** | `customer_unique_id` (o `customer_id` muda a cada pedido no dataset da Olist) |

## Métricas

| Métrica | Fórmula | Universo | Observações |
|---|---|---|---|
| **GMV** | Σ (`price` + `freight_value`) dos itens | pedidos válidos | inclui frete; não usa `payment_value` (que inclui juros de parcelamento) |
| **Ticket médio** | GMV ÷ nº de pedidos válidos | pedidos válidos | por pedido, não por item |
| **Taxa de cancelamento** | pedidos `canceled` ÷ total de pedidos criados | todos os pedidos | `unavailable` é reportado à parte (indisponibilidade ≠ cancelamento) |
| **% entregas no prazo** | entregues no prazo ÷ pedidos entregues | pedidos entregues | comparação por dia, não por horário |
| **Lead time de entrega** | dias entre `order_purchase_timestamp` e `order_delivered_customer_date` | pedidos entregues | reportar média **e** mediana (distribuição tem cauda longa) |
| **Review score médio** | média de `review_score` (1–5) | reviews de pedidos entregues | se o pedido tiver mais de uma review, vale a mais recente (`review_answer_timestamp`) |
| **Taxa de recompra** | clientes únicos com 2+ pedidos válidos ÷ clientes únicos com 1+ pedido válido | pedidos válidos | usa `customer_unique_id` |
| **Concentração de vendas** | GMV dos 10 vendedores de maior GMV ÷ GMV total | pedidos válidos | calculada por período |

## Recortes padrão

Todas as métricas podem ser recortadas por: período (dia/mês da compra), categoria de produto (traduzida para inglês), vendedor, estado (UF) do cliente e do vendedor.
