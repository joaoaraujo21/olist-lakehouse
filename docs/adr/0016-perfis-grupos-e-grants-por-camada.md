# ADR 0016 — Perfis, grupos e `GRANT` por camada

- **Status:** aceito
- **Data:** 2026-11-18
- **Fase:** 6 — Governança e LGPD

## Contexto

Até a Fase 5 todo objeto é lido pelo dono do catálogo. A Fase 6 precisa separar quem consome (analista, gestor), quem trata dado pessoal (DPO) e quem opera (engenharia). O Unity Catalog só concede permissão a **grupos de conta**: um teste em 2026-11-18 mostrou que `GRANT` a um grupo da conta funciona e a um grupo local do workspace (`admins`) falha com `PRINCIPAL_DOES_NOT_EXIST`.

## Decisão

1. **Quatro grupos novos**, criados na conta e específicos do projeto: `olist_analista`, `olist_gestor`, `olist_dpo` e `olist_engenharia`. Grupos já existentes no workspace não são reaproveitados.
2. `olist_engenharia` existe para **isentar o dono do pipeline** das máscaras sem escrever e-mail de usuário no código: as materialized views da gold são gravadas pela identidade que roda o pipeline, e uma máscara aplicada a ela gravaria o dado já mascarado.
3. **Acesso por camada, o menor possível:**

| Camada | analista | gestor | DPO | engenharia |
|---|---|---|---|---|
| `landing`, `bronze`, `silver` | nenhum | nenhum | leitura | dono |
| `gold` (tabelas, views e Metric Views) | leitura | leitura | leitura | dono |
| `quality` e `ops` | só `quality.validacao_*` | só `quality.validacao_*` | leitura | dono |

4. Na gold, o `SELECT` é concedido **objeto a objeto**, e não no schema inteiro. O schema guarda tabelas de apoio das materialized views (`__materialization_*`) e o log do pipeline, que não passam pelas máscaras; fora da lista concedida, ficam inacessíveis aos consumidores.
5. O notebook de grants é **idempotente** e roda de novo depois de cada objeto novo na gold.

## Alternativas consideradas

1. **`SELECT` no schema `gold`.** Mais simples, mas herdaria acesso às tabelas de apoio sem máscara.
2. **Reaproveitar os grupos existentes.** Evita criar grupos, mas mistura permissões de outras finalidades.
3. **Isentar o dono pelo e-mail.** Põe dado pessoal no repositório público.

## Consequências

- (+) Quem consome só enxerga a gold; o cadastro da silver fica fora do alcance.
- (+) A isenção da engenharia é um grupo: trocar o responsável não exige mexer em função.
- (−) A demonstração dos perfis depende de alternar a participação nos grupos, porque só há um usuário humano.
- (−) O dono e os administradores têm acesso por privilégio próprio: a ausência de acesso à silver é provada por `SHOW GRANTS`, e não por consulta do dono.
