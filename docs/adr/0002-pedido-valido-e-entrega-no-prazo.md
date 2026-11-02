# ADR 0002 — Definição de "pedido válido" e "entregue no prazo"

- **Status:** aceito (revisar ao final da Fase 2, com os resultados de qualidade)
- **Data:** 2026-11-02
- **Fase:** 0 — Fundação

## Contexto

Comercial, Logística e Financeiro calculam GMV e "pedido entregue" de formas diferentes (problema central do Escopo de Negócio). Cinco dos oito KPIs oficiais dependem de dois conceitos: **pedido válido** (GMV, ticket médio, taxa de recompra, concentração de vendas) e **entregue no prazo** (% de entregas no prazo, lead time).

O campo `order_status` da tabela de pedidos da Olist tem 8 valores. Distribuição aproximada no dataset original (a confirmar na Fase 2):

| `order_status` | Pedidos | Significado |
|---|---:|---|
| `delivered` | ~96.478 | entregue ao cliente |
| `shipped` | ~1.107 | despachado, ainda em trânsito |
| `canceled` | ~625 | cancelado |
| `unavailable` | ~609 | sem estoque/indisponível — na prática, não concretizado |
| `invoiced` | ~314 | faturado |
| `processing` | ~301 | em separação |
| `created` | ~5 | criado, ainda não aprovado |
| `approved` | ~2 | pagamento aprovado |

Anomalias conhecidas que as definições precisam tratar: pedidos sem nenhum item (concentrados em `unavailable`/`canceled`), pedidos `delivered` sem data de entrega ao cliente, e pedidos cancelados que possuem datas de entrega preenchidas.

## Decisão

### Pedido válido

> Um pedido é **válido** quando `order_status` **não** é `canceled` nem `unavailable` **e** o pedido possui **ao menos um item** em `order_items`.

- Inclui pedidos em andamento (`created`, `approved`, `processing`, `invoiced`, `shipped`) além dos `delivered`: GMV mede venda realizada no marketplace, não entrega concluída.
- O status considerado é o **status atual** do pedido. Quando um pedido válido é cancelado depois, ele sai do GMV do período da compra (o número do passado pode mudar — isso é esperado e documentado).
- O período de referência de um pedido é a data de compra (`order_purchase_timestamp`).

### Pedido entregue

> Um pedido é **entregue** quando `order_status = 'delivered'` **e** `order_delivered_customer_date` não é nulo.

Pedidos `delivered` sem data de entrega não entram nas métricas de entrega e são sinalizados na camada `quality`.

### Entregue no prazo

> Um pedido entregue está **no prazo** quando `DATE(order_delivered_customer_date) <= DATE(order_estimated_delivery_date)`.

- Comparação por **data (dia)**, não por horário: a data estimada é informada ao cliente como um dia, sem hora (vem com `00:00:00`). Entregar em qualquer horário do dia estimado conta como no prazo.
- Denominador de "% entregas no prazo": pedidos **entregues** (definição acima), não todos os pedidos.
- Pedidos ainda não entregues cuja data estimada já passou **não** entram nesse KPI; eles são o objeto do monitor de pedidos em risco/atrasados (Fase 5).

### Regras de qualidade derivadas (Fase 2)

| Situação | Tratamento |
|---|---|
| Pedido sem itens | fica fora de "pedido válido"; contado no painel de qualidade |
| `delivered` sem `order_delivered_customer_date` | fora das métricas de entrega; quarentena/alerta |
| Data de entrega anterior à data de compra | data impossível → quarentena |
| `canceled`/`unavailable` com data de entrega | mantém o status como verdade; sinalizado no painel de qualidade |

Todos os timestamps do dataset estão em horário local do Brasil, sem fuso; serão tratados como `America/Sao_Paulo`.

## Alternativas consideradas

1. **Válido = apenas `delivered`.** Número mais "seguro" para o Financeiro, mas subestima a venda e atrasa o GMV em dias/semanas (só conta após a entrega). Pode existir como métrica auxiliar ("GMV entregue"), não como GMV oficial.
2. **Válido = pagamento aprovado (`order_approved_at` não nulo).** Próximo da prática de mercado, mas há pedidos `delivered` sem data de aprovação no dataset; a regra excluiria vendas reais por falha de registro.
3. **Prazo comparando timestamp completo.** Penalizaria entregas feitas durante o dia estimado (qualquer horário > 00:00), inflando artificialmente o atraso.
4. **Denominador de prazo = todos os pedidos despachados.** Mistura pedidos ainda em trânsito, cujo desfecho não é conhecido; o KPI oscilaria conforme o volume em trânsito.

## Consequências

- (+) Uma definição única, escrita e versionada, para os 8 KPIs; o dicionário de métricas (`docs/dicionario_metricas.md`) referencia este ADR.
- (+) As anomalias viram regras de qualidade explícitas em vez de filtros escondidos em consultas.
- (−) O GMV de períodos passados pode mudar quando pedidos são cancelados depois; relatórios fechados precisarão de snapshot se isso virar requisito.
- (−) As contagens da tabela de contexto são aproximadas e devem ser confirmadas na Fase 2.
