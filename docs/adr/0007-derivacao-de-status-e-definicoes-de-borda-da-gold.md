# ADR 0007 — Derivação do status do pedido e definições de borda da gold

- **Status:** aceito
- **Data:** 2026-11-15
- **Fase:** 4 — Gold e Métricas

## Contexto

O ADR 0002 e o dicionário v0 definiram "pedido válido" e "entregue" sobre as colunas `order_status` e `order_delivered_customer_date` do dataset original. Na silver essas colunas não existem: o status e as datas de entrega vivem em `order_status_events` (um evento por mudança de status). Além disso, os dados têm casos de borda que precisam de uma regra única para SQL, Metric View e dashboard darem o mesmo número:

- 2 pedidos sem nenhum evento (ambos com 1 item);
- 745 pedidos sem item;
- 544 pedidos com mais de uma review e 17 reviews sem pedido;
- 1.370 pedidos com `event_ts` fora da ordem do `event_sequence` (127 deles no evento final); só 223 têm falha registrada pelo simulador, a maior parte vem do dataset original.

## Decisão

1. **Status atual** = `status` do evento de maior `event_sequence`. **Data de entrega** = `event_ts` do evento `delivered` (no máximo um por pedido). Pedido sem evento tem status `desconhecido`.
2. **Pedido válido** segue o ADR 0002 literalmente: ao menos 1 item e status diferente de `canceled` e `unavailable`. O pedido `desconhecido` com item é válido.
3. **Pedido sem item** não é válido e conta no total de pedidos criados da taxa de cancelamento.
4. **Review mais recente** por pedido: maior `review_answer_timestamp`, desempate por `review_creation_date` e `review_id`. Review sem pedido fica fora das métricas.
5. **Sem conversão de fuso**: os timestamps da silver já estão no relógio local do Brasil.
6. **Recompra** usa o `customer_unique_id` da versão do cadastro vigente na compra; o cliente excluído continua contando no passado.
7. **Concentração de vendas** é definida por mês (top 10 do mês), com desempate por `seller_id`.
8. As definições vivem em **Metric Views** (`mv_pedidos`, `mv_itens`, `mv_concentracao`), e a verificação recalcula os KPIs diretamente da silver.

## Alternativas consideradas

1. **Ordenar os eventos por `event_ts`.** Mudaria o status de pedidos por causa de uma inversão de relógio que vem do próprio dado. A sequência é a ordem do processo e não depende de relógio.
2. **Tratar o pedido `desconhecido` como não válido.** Mais conservador, mas muda a regra do ADR 0002 por causa de 2 pedidos; a regra literal é mais simples de explicar e auditar.
3. **Top 10 do período inteiro para a concentração.** Número único, mas não responde "por período", que o dicionário v0 já pedia; o mês permite série histórica.
4. **Definir os KPIs só no dashboard.** Cada ferramenta teria sua cópia da fórmula, que é o problema que o projeto resolve.

## Consequências

- (+) Uma definição por métrica, versionada no dicionário v1 e executada pelas Metric Views.
- (+) A validação independente pode recalcular os números sem usar o código da gold.
- (−) O status final depende da qualidade do `event_sequence`; um evento com sequência errada mudaria o status. A silver já valida a sequência.
- (−) O pedido `desconhecido` entra no GMV. Se a Fase 5 ou 6 mostrar que o caso é frequente, a regra deve ser revista numa nova versão do dicionário.
- (−) A taxa de recompra por mês mistura "comprou no mês" com "comprou 2+ vezes em qualquer período"; isso está explícito no dicionário.
