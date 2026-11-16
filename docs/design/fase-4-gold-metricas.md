# Documento de Desenho Técnico — Fase 4: Gold e Métricas

| | |
|---|---|
| **Projeto** | Olist Lakehouse |
| **Fase** | 4 — Gold e Métricas |
| **Responsável técnico** | João Guilherme Araujo |
| **Data** | 2026-11-13 |
| **Status** | Implementado (aprovado em 2026-11-15; fase concluída em 2026-11-16) |
| **Documentos relacionados** | [Desenho da Fase 3](fase-3-cdc-scd2.md) · [ADR 0002 — Pedido válido e entrega no prazo](../adr/0002-pedido-valido-e-entrega-no-prazo.md) · [ADR 0006 — Histórico SCD2](../adr/0006-historico-scd2-do-cadastro.md) · [Dicionário de métricas v0](../dicionario_metricas.md) |

---

## 1. Objetivo deste documento

Registrar o desenho da camada gold **antes** da implementação: o modelo dimensional, como cada um dos 8 KPIs é calculado, como o cadastro vigente na data do pedido entra nos fatos, onde a gold é construída e como provaremos que SQL, Metric View e dashboard dão o mesmo número.

## 2. Contexto e alinhamento com o negócio

Hoje Comercial, Logística e Financeiro calculam GMV e "pedido entregue" cada um do seu jeito. A silver já entrega dado confiável e o histórico de cadastro, mas ainda é um conjunto de tabelas técnicas: o status do pedido está espalhado em eventos, as avaliações podem ser duplicadas e o cadastro tem versões. A gold transforma isso em **uma definição por métrica**, consumível por qualquer pessoa.

| Área | Dor | O que a Fase 4 entrega |
|---|---|---|
| Comercial | não sabe quais vendedores e categorias performam | `gold_seller_scorecard` e ranking de categorias |
| Logística | atrasos descobertos tarde | `gold_delivery_sla` (o monitor em tempo quase real é da Fase 5) |
| Atendimento (CX) | nota baixa sem causa clara | relação entre atraso e review score |
| Financeiro | GMV sem fonte única | métricas oficiais em Metric Views |
| Diretoria | sem visão consolidada | dashboard executivo |

**Critérios de aceite** (épicos E3 e E4 do escopo): os 8 KPIs reproduzíveis por SQL e por dashboard **com os mesmos números**; scorecard com atraso, cancelamento e review por vendedor, com o cadastro do vendedor **na data da venda**.

## 3. Estado de partida (dados em 2026-11-13)

Consultado nas tabelas silver:

| Fato | Valor |
|---|---|
| Pedidos (`silver.orders`) | 98.476, com `customer_id`, data da compra e data estimada de entrega; **sem** `order_status` nem datas de entrega |
| Status e datas de entrega | vêm de `silver.order_status_events` (98.474 pedidos têm eventos; **2 pedidos não têm nenhum**) |
| Status final do pedido (último evento por `event_sequence`) | `delivered` 93.247 · `shipped` 2.628 · `approved` 657 · `unavailable` 608 · `canceled` 584 · `invoiced` 314 · `processing` 301 · `created` 135 |
| Pedidos sem nenhum item | 745 |
| Itens: chave repetida / sem pedido / sem vendedor / sem produto | 0 · 0 · 0 · 0 |
| Pedidos com item / vendedores com venda | 97.731 / 3.077 |
| GMV bruto (Σ preço + frete, todos os itens) | R$ 15.722.601,92 |
| Reviews sem pedido na silver | 17 |
| Pedidos com mais de uma review | 544 |
| Produtos sem tradução de categoria | 623: 610 sem categoria e 13 de 2 categorias que a tradução não cobre |
| Clientes únicos com mais de um `customer_id` | 2.933 |
| Pedidos de clientes excluídos / itens de vendedores excluídos | 288 / 280 (cadastro encontrado pela versão vigente na compra: 98.476 de 98.476 pedidos) |
| Pedidos com `event_ts` fora da ordem do `event_sequence` | 127 (61 `delivered` com `approved` mais tarde que a entrega, 43 `shipped` com `approved` mais tarde, 23 `delivered` com `shipped` mais tarde) |
| Período das compras | 2026-09-06 a 2028-08-21 (simulação avançada; consistente com a Fase 3) |

O que isso muda no desenho:
- **Não existe `order_status` pronto.** O estado atual do pedido precisa ser derivado dos eventos, e o ADR 0002 e o dicionário v0 pressupõem as colunas originais da Olist (`order_delivered_customer_date`). A gold tem que reconstruí-las.
- Em 127 pedidos o `event_ts` não segue a sequência dos eventos (a aprovação aparece depois da expedição ou da entrega). É preciso decidir qual dos dois ordena (G2).
- 745 pedidos sem item e 17 reviews órfãs são casos de borda que as definições precisam cobrir.
- Os 2 pedidos sem nenhum evento têm 1 item cada: pela regra do ADR 0002 (status não é `canceled` nem `unavailable` e há item) são **válidos**, com status `desconhecido`. A etapa 4.2 fixa isso no dicionário v1.
- O `review_id` se repete em 795 linhas (94.458 ids para 95.253 linhas), mas o par (`review_id`, `order_id`) é único: é a granularidade de `fact_reviews`.
- Das 71 categorias traduzidas, 2 categorias de produto não têm tradução (23 itens): a `dim_product` usa o nome em português nesses casos e marca a origem em `categoria_origem`.

## 4. Escopo

**Dentro do escopo:** modelo dimensional (`dim_date`, `dim_product`, `dim_customer`, `dim_seller`, `fact_orders`, `fact_order_items`, `fact_reviews`); agregados (`gold_seller_scorecard`, `gold_delivery_sla`, `gold_daily_kpis`); os 8 KPIs como Metric Views; verificação independente dos números; dashboard executivo e scorecard; documentação, ADRs e versão 1 do dicionário de métricas.

**Fora do escopo:** `gold_orders_live` e o monitor de risco de atraso (Fase 5); mascaramento, perfis de acesso e exclusão auditável (Fase 6); testes unitários, CI e orquestração em job (Fase 7); ajuste fino de performance (Fase 8).

## 5. Visão geral da solução

```
silver.orders / order_items / order_payments / order_reviews / order_status_events / products / category_translation
silver.customers_historico / sellers_historico  (SCD2, Fase 3)
      │
      ├──► dim_date, dim_product, dim_customer (versões), dim_seller (versões)
      ├──► fact_orders        (1 linha por pedido: status, datas, flags, GMV, lead time, review, versão do cliente na compra)
      ├──► fact_order_items   (1 linha por item: preço, frete, produto, vendedor, versão do vendedor na compra)
      ├──► fact_reviews       (1 linha por review, com marca da mais recente por pedido)
      └──► gold_seller_scorecard · gold_delivery_sla · gold_daily_kpis
                 │
                 └──► Metric Views (8 KPIs) ──► AI/BI Dashboard · Genie · SQL
Validação independente: quality.validacao_gold (KPIs recalculados por outro caminho)
```

## 6. Perguntas e respostas de desenho

#### G1. Onde a gold é construída e com que objetos?
**Proposta:** *materialized views* declaradas em um **pipeline próprio** (`olist.gold`), lendo as tabelas silver; as Metric Views e o dashboard ficam sobre essas tabelas.
**Por quê:** a gold é agregação e junção sobre a silver já pronta, que é exatamente o que uma materialized view faz, com atualização incremental e dependências resolvidas pelo pipeline; manter a gold separada da silver deixa claro qual execução quebrou o quê.
**Ponto de atenção:** o Free Edition limita os pipelines ativos, e o workspace já tem outro pipeline que não é deste projeto. Se a criação de um segundo for recusada, o **plano reserva** é um terceiro arquivo-fonte (`src/gold/...`) no pipeline `olist.silver`. O teste é uma criação e um *Dry run* na etapa 4.1.
**Alternativa:** tabelas Delta escritas por notebook com `MERGE`. Mais controle, mas reimplementa o que a materialized view já faz e perde a ordem de dependências.

#### G2. Como se define o status atual e as datas de entrega do pedido?
**Proposta:** o status atual é o do evento de **maior `event_sequence`** do pedido; `order_delivered_customer_date` é o `event_ts` do evento `delivered` (existe no máximo um por pedido). Pedido sem evento (2) fica com status `desconhecido` e fora das métricas de entrega.
**Por quê:** `event_sequence` é a ordem do processo (criado, aprovado, expedido, entregue) e não depende de relógio; o `event_ts` tem inversões nas 127 linhas descritas (verificar na etapa 4.1 se vêm da origem). O ADR 0002 já fixa que vale o status **atual**.
**Alternativa:** ordenar por `event_ts`. Mudaria o status de 127 pedidos por causa de uma inversão de relógio.

#### G3. Qual o modelo dimensional e a granularidade?
**Proposta:**

| Tabela | Granularidade | Observação |
|---|---|---|
| `fact_order_items` | 1 linha por item | preço, frete, produto, vendedor; versão do vendedor vigente na compra |
| `fact_orders` | 1 linha por pedido | status atual, datas, flags (`pedido_valido`, `entregue`, `entregue_no_prazo`), GMV, lead time, nota mais recente, versão do cliente na compra |
| `fact_reviews` | 1 linha por review | `eh_mais_recente` marca a review que vale por pedido; reviews sem pedido ficam com `pedido_existe = false` |
| `dim_date` | 1 linha por dia | de 2026-09-06 a 2028-08-21, mais folga |
| `dim_product` | 1 linha por produto | categoria em inglês; 610 sem categoria viram `desconhecida` e 13 sem tradução mantêm o nome em português |
| `dim_customer` | 1 linha por **versão** | de `customers_historico`, com `customer_unique_id` |
| `dim_seller` | 1 linha por **versão** | de `sellers_historico` |

**Por quê:** a granularidade de item é a das métricas de receita; a de pedido é a das de entrega, cancelamento e review; separar evita contar frete ou GMV duas vezes.

#### G4. Como o fato aponta para o cadastro da data do pedido?
**Proposta:** as dimensões de cliente e vendedor têm uma **chave substituta por versão** (`customer_sk`, `seller_sk`, um hash de `id` + `__START_AT`), e o fato guarda a chave da versão vigente em `order_purchase_timestamp` (junção por intervalo, feita uma vez na construção, com `>= __START_AT` e `< __END_AT`, como no ADR 0006). Quem consome faz uma junção simples por `_sk`.
**Por quê:** a junção por intervalo é a parte difícil; feita uma vez, o dashboard e o Genie não precisam repeti-la. O pedido feito exatamente no instante da mudança fica com a versão nova (convenção do ADR 0006), documentada.
**Cadastro excluído hoje:** os 288 pedidos de clientes excluídos e os 280 itens de vendedores excluídos encontram versão (fechada), então não há pedido sem dono. A coluna `cadastro_excluido_hoje` registra isso para a Fase 6.

#### G5. Como ficam os 8 KPIs? (dicionário v1)
**Proposta:** manter as definições do dicionário v0 e fixar o que ficou em aberto, numa **versão 1** do dicionário (e um ADR curto):

| Questão em aberto | Proposta |
|---|---|
| Status e data de entrega não existem como colunas | derivados dos eventos (G2) |
| Pedido sem item (745) | não é pedido válido (já na definição); entra no total de pedidos criados da taxa de cancelamento |
| Mais de uma review por pedido (544) | vale a de maior `review_answer_timestamp` (já no dicionário); desempate por `review_creation_date` |
| Review sem pedido (17) | fica fora de todas as métricas, sinalizada |
| Fuso horário | os timestamps da silver já estão no relógio local do Brasil; **não converter** de novo (confirmar na etapa 4.1) |
| Recompra e cliente excluído | contar pelo `customer_unique_id` da versão vigente na compra: o cliente excluído continua contando no passado, e a Fase 6 decide sobre anonimização |
| Datas até 2028 | o período de referência é a data da compra; os gráficos usam o intervalo real dos dados, sem "hoje" |

**Por quê:** o épico E3 exige o mesmo número em SQL e dashboard; cada interpretação em aberto é uma fonte de divergência.

#### G6. Metric Views e dashboard
**Proposta:** as 8 métricas viram **Metric Views** (definição única, com os recortes padrão: período, categoria, vendedor, UF do cliente e do vendedor) sobre as tabelas gold; o AI/BI Dashboard e o Genie consomem as Metric Views.
**Ponto de atenção:** confirmar na etapa 4.2 que Metric Views, AI/BI Dashboard e Genie estão disponíveis no Free Edition. Se algum não estiver, o plano reserva é uma view SQL com as mesmas definições e o dashboard sobre ela.

#### G7. Como provar que os números batem?
**Proposta:** notebook `quality/04_validacao_gold` com a mesma tabela de resultados (`quality.validacao_gold`): os 8 KPIs são **recalculados por outro caminho** (direto das tabelas silver, sem usar as tabelas gold) e comparados com a Metric View e com as consultas do dashboard; mais verificações de granularidade (uma linha por pedido e por item), chaves substitutas (nenhum fato sem dimensão) e totais (soma dos itens = GMV).
**Por quê:** é a mesma lógica da Fase 2: a gold não pode ser auditada pelo próprio código que a produz.

#### G8. Liquid clustering e as chaves
**Proposta:** declarar `CLUSTER BY` nas duas tabelas de fato (data da compra e vendedor/cliente). O volume é pequeno (cerca de 100 mil linhas), então o ganho é educativo; a medição e o ajuste ficam para a Fase 8.
**Alternativa:** não clusterizar agora. Mais simples, mas deixa o tema do liquid clustering para depois sem exercício.

## 7. Resumo das decisões

| # | Decisão | Status |
|---|---|---|
| G1 | Materialized views em pipeline próprio `olist.gold` (reserva: terceiro arquivo-fonte do `olist.silver`) | implementada |
| G2 | Status atual = maior `event_sequence`; data de entrega = `event_ts` do evento `delivered` | implementada |
| G3 | Fatos de item, pedido e review; dimensões de data, produto, cliente e vendedor (as duas últimas por versão) | implementada |
| G4 | Chave substituta por versão, junção por intervalo uma vez na construção | implementada |
| G5 | Dicionário v1 fecha as pontas em aberto (derivação, pedido sem item, review duplicada ou órfã, fuso, recompra) | implementada |
| G6 | 8 KPIs em Metric Views; dashboard e Genie sobre elas | implementada |
| G7 | Validação independente dos KPIs (`quality.validacao_gold`) | implementada |
| G8 | `CLUSTER BY` nas tabelas de fato | implementada |

## 8. Riscos e pontos de atenção

- **Limite de pipelines no Free Edition:** pode forçar a gold para dentro do pipeline `olist.silver` (G1).
- **Disponibilidade de Metric Views, Dashboard e Genie** no Free Edition (G6).
- **Inversões de `event_ts` (127):** documentadas, não corrigidas; a verificação da etapa 4.1 diz se vêm da origem ou do simulador.
- **O número do passado pode mudar:** um pedido válido cancelado depois sai do GMV do mês da compra (ADR 0002). É esperado e vai no dicionário.
- **Cota do compute:** as materialized views reprocessam bem pouco por execução; evitar *full refresh* sem necessidade.
- **Datas até 2028:** nenhum filtro pode depender de "hoje".

## 9. Plano de implementação e critérios de aceite

| Etapa | Entrega |
|---|---|
| 4.1 | Pipeline `olist.gold` (ou reserva) com `dim_date`, `dim_product`, `dim_customer`, `dim_seller`, `fact_order_items`, `fact_orders` e `fact_reviews`; verificações de granularidade e de chave substituta |
| 4.2 | Dicionário v1 e os 8 KPIs em SQL e Metric Views, com a validação independente (`quality.validacao_gold`) |
| 4.3 | `gold_seller_scorecard`, `gold_delivery_sla`, `gold_daily_kpis` e a relação atraso × review |
| 4.4 | Dashboard executivo e scorecard, Genie, tags e comentários, ADRs, documentação e encerramento |

**Critérios de aceite** (a detalhar por etapa):

| # | Verificação | Esperado |
|---|---|---|
| 1 | `fact_orders` = pedidos da silver | 98.476 linhas, uma por pedido |
| 2 | `fact_order_items` = itens da silver | 111.592 linhas, uma por item |
| 3 | Fatos sem dimensão (chave substituta nula) | 0 |
| 4 | Soma de preço + frete dos itens de pedidos válidos | igual ao GMV da Metric View |
| 5 | Os 8 KPIs: SQL × Metric View × dashboard | números idênticos |
| 6 | Scorecard: cadastro do vendedor na data da venda | 265 vendedores que se mudaram e venderam; pelo instante da compra, 4.022 itens antes e 7.231 depois da primeira mudança (a Fase 3 contou 3.906 e 7.347 pelo limite de envio; o total, 11.253, é o mesmo; ADR 0008) |
| 7 | Execução repetida sem dados novos | contagens iguais |
