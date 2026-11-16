# ADR 0008 — Scorecard de vendedores: pedidos compartilhados e cadastro na data da compra

- **Status:** aceito
- **Data:** 2026-11-15
- **Fase:** 4 — Gold e Métricas

## Contexto

O scorecard responde "como cada vendedor performa em GMV, cancelamento, entrega no prazo e review". Duas coisas complicam a conta:

- **Pedidos com mais de um vendedor.** 1.256 pedidos (1,3%) têm itens de vendedores diferentes. O status, o atraso e a review pertencem ao **pedido**, não ao item. Somando por vendedor, o mesmo pedido aparece mais de uma vez: são 99.049 pares (pedido, vendedor) para 97.731 pedidos com item.
- **Cadastro que muda.** 266 vendedores mudaram de endereço. Atribuir as vendas à cidade atual colocaria vendas antigas no lugar errado (ADR 0006).

O desenho da Fase 4 também previa o critério "3.906 itens antes e 7.347 depois da mudança de endereço", vindo da Fase 3. Esses números usavam o limite de envio (`shipping_limit_date`) como data da venda. A gold usa o instante da compra, como o resto do projeto.

## Decisão

1. **Grão do scorecard:** uma linha por **versão do cadastro do vendedor** (`seller_sk`), com a cidade, a UF e o intervalo de validade da versão. Cada venda cai na versão vigente no instante da compra.
2. **Pedidos compartilhados (opção A):** o pedido entra no scorecard de **cada** vendedor que tem item nele, com o status, o atraso e a review do pedido inteiro. A coluna `pedidos_compartilhados` conta os que têm mais de um vendedor. Por isso a soma de `pedidos` no scorecard (99.049) passa do total de pedidos com item.
3. **O GMV não duplica:** soma apenas os itens do próprio vendedor, em pedidos válidos. A soma do GMV do scorecard é igual ao GMV oficial.
4. **Critério 6 reescrito:** pelo instante da compra, nos vendedores que se mudaram, são **4.022 itens antes** da primeira mudança (102 vendedores) e **7.231 depois** (213 vendedores), 265 vendedores com venda. O total (11.253) é o mesmo da Fase 3; os 116 itens de diferença foram comprados antes da mudança, mas tinham limite de envio depois dela.

## Alternativas consideradas

1. **Só pedidos de vendedor único nas métricas de entrega e review (opção B).** Sem ambiguidade, mas descarta 1.256 pedidos e faz o vendedor parecer melhor ou pior conforme a fração que sobra.
2. **Dividir o pedido entre os vendedores** (peso pelo valor dos itens). Soma fecha no total, mas atraso e review não se dividem: seria atribuir meia nota a cada um.
3. **Cadastro atual do vendedor** (sem versão). Mais simples, mas contradiz o objetivo do scorecard e o ADR 0006.
4. **Manter o limite de envio como data da venda.** Reproduz os números da Fase 3, mas diverge da data de compra usada em todas as métricas.

## Consequências

- (+) Cada vendedor é avaliado por tudo o que passou por ele, e o resultado é fácil de explicar.
- (+) As vendas ficam na cidade que o vendedor tinha na data da compra.
- (−) `SUM(pedidos)` do scorecard não é o total de pedidos: quem somar vendedores para obter o total de pedidos conta 1,3% a mais. Isso está no dicionário e na descrição da tabela.
- (−) Um atraso causado por um vendedor pesa também sobre o outro vendedor do mesmo pedido.
- (−) Quem comparar com os números da Fase 3 verá 4.022 e 7.231, não 3.906 e 7.347; o motivo está registrado aqui.
