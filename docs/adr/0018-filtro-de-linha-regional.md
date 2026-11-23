# ADR 0018 — Filtro de linha regional

- **Status:** aceito (implementação na etapa 6.3)
- **Data:** 2026-11-18
- **Fase:** 6 — Governança e LGPD

## Contexto

O gestor de uma área regional precisa dos números da sua região sem enxergar os pedidos das demais. A máscara de coluna não resolve: o recorte é por linha.

## Decisão

1. O gestor vê só os pedidos das **UFs da sua região**, definidas na tabela `ops.acesso_regional` (grupo → UFs). A função do filtro consulta a tabela; mudar o recorte não exige alterar a função.
2. Aplicado com `ALTER ... SET ROW FILTER` em `gold.fact_orders` e `gold.dim_customer`. Analista, DPO e engenharia veem todas as linhas.
3. **Não se filtram** os agregados que somam todas as UFs (`gold_daily_kpis` e afins): não têm dado pessoal, e o dicionário define o número oficial como o total.

## Alternativas consideradas

1. **Lista de UFs dentro da função.** Funciona, mas cada mudança de recorte vira alteração de código.
2. **Uma view por região.** Multiplica objetos e deixa a tabela de base legível.

## Consequências

- (+) O recorte é dado, e não código.
- (−) Um filtro por tabela; materialized views seguem o cuidado de isenção do ADR 0016.
- (−) O gestor regional vê, nos KPIs filtrados, números diferentes dos oficiais (por construção); o dicionário registra essa diferença na etapa 6.5.
