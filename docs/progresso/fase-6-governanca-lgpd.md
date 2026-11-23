# Fase 6 — Governança e LGPD

- **Período:** iniciada em 2026-11-18
- **Status:** 🟡 em andamento — 6.1 (desenho) aprovada em 2026-11-18; 6.2 em execução (o João sobe e executa os notebooks)
- **Branch/PRs:** uma branch e um PR por etapa; 6.1 em `fase-6.1/desenho-governanca`

## Objetivo

**Negócio:** a empresa precisa mostrar que trata dado pessoal com controle: quem pode ver o quê, o que fica mascarado e como um cliente é excluído de ponta a ponta, com evidência auditável.

**Técnico:** Unity Catalog como camada de governança: grants por perfil (analista, gestor, DPO), máscaras de coluna e filtros de linha por categoria de dado pessoal, tags de PII e exclusão de cliente ponta a ponta (`DELETE`, Change Data Feed e `VACUUM`) com evidência.

## Ponto de partida

- Tags de PII (`pii`, `pii_categoria`: `identificador` e `localizacao`) já existem na silver, na gold e nos históricos SCD2 (Fases 2 a 4); falta aplicar máscaras e perfis por categoria.
- `dim_customer`, `dim_seller`, `fact_orders.customer_unique_id` e o scorecard têm dado pessoal e guardam endereços antigos.
- Os objetos da visão ao vivo (`gold_orders_live`, `vw_pedidos_em_risco`, `ops.streaming_batches`) não têm dado pessoal (`contem_pii = false`).
- O teste de consumo no Power BI (Fase 5) importou tabelas com dado pessoal e mostrou que a separação por área exige relatórios ou apps por público, já que a segurança por linha só filtra dados.

## Pendências herdadas

- **Exclusão de cliente e o estado ao vivo:** a leitura em streaming ignora `DELETE` na bronze (ADR 0015); o procedimento de exclusão precisa reconstruir `gold_orders_live` ou apagar o pedido nele.
- **Localização do vendedor:** a Fase 4 classificou cidade, UF e CEP do cadastro do vendedor como PII `localizacao`; a Fase 6 decide se mascara (é a localização da empresa).
- **Perfis no Free Edition:** a Fase 0 já registrou as limitações de grants; verificar o que o ambiente permite antes de desenhar os perfis.

## Etapas

Desenho completo em [docs/design/fase-6-governanca-lgpd.md](../design/fase-6-governanca-lgpd.md).

| Etapa | Entrega | Branch | Status |
|---|---|---|---|
| 6.1 | Desenho técnico e decisões G1 a G8 (perfis, máscaras, filtro de linha, exclusão, evidência) | `fase-6.1/desenho-governanca` | ✅ aprovada (ADRs 0016 a 0019) |
| 6.2 | Grupos, `GRANT` por camada, classificação das lacunas e máscaras de coluna | `fase-6.2/perfis-e-mascaras` | a iniciar |
| 6.3 | Filtro de linha regional | `fase-6.3/filtros-de-linha` | a iniciar |
| 6.4 | Exclusão de cliente ponta a ponta com evidência | `fase-6.4/exclusao-de-cliente` | a iniciar |
| 6.5 | Auditoria e lineage, validação, documentação e encerramento | `fase-6.5/auditoria-e-encerramento` | a iniciar |

## O que foi feito

### 6.1 — Desenho (2026-11-18)

Levantamento somente de leitura do ambiente:

- 25 colunas com tag de PII em 8 tabelas da silver e da gold; nenhum `GRANT` concedido; todos os schemas com o mesmo dono.
- **Lacunas de classificação:** bronze (`customers`, `sellers`, `order_reviews`), `silver.order_reviews` (texto livre) e as quarentenas `customers_quarentena` e `order_reviews_quarentena` guardam dado pessoal sem tag (G8).
- 99.291 `customer_id` para 95.695 pessoas (`customer_unique_id`): a exclusão de uma pessoa remove várias linhas.
- Tabelas com deletion vectors: o `DELETE` sozinho não remove o dado dos arquivos; o procedimento de exclusão inclui `REORG ... APPLY (PURGE)` e `VACUUM`.
- `system.access.table_lineage` e `system.access.audit` estão acessíveis: servem de lineage e de trilha.
- `CREATE FUNCTION` funciona no schema `ops`; não há escopo de secrets; só há 1 usuário, então a demonstração dos dois perfis alterna a participação nos grupos.

- **Grants só valem para grupos de conta:** `GRANT` a um grupo da conta funcionou em tabela de teste; a um grupo local do workspace (`admins`) falhou com `PRINCIPAL_DOES_NOT_EXIST`. Os grupos do projeto precisam ser criados como grupos de conta.
- **Tabelas de apoio na gold:** o schema guarda `__materialization_*` (cópia física de cada materialized view, sem máscara) e o log do pipeline; por isso o `SELECT` é concedido objeto a objeto (ADR 0016).

Resultado: documento de desenho com 8 decisões, aprovadas em 2026-11-18 na opção recomendada (criar os grupos; matriz de máscaras; vendedor sem máscara; apagar o cadastro e manter o pedido anônimo) e registradas nos ADRs 0016 a 0019. Ajuste: quarto grupo, `olist_engenharia`, para isentar o dono do pipeline das máscaras.

### 6.2 — Perfis, grants, classificação e máscaras (em execução)

Mudança de método: a partir desta fase os notebooks são entregues prontos e o João sobe, executa e testa. Dois notebooks em `src/governanca/`: `01_perfis_e_grants` (verifica os grupos, concede o acesso por camada e registra verificações em `quality.validacao_governanca`) e `02_classificacao_e_mascaras` (completa as tags de PII, cria as funções de máscara e as aplica na gold guiado pelas tags). Resultado a registrar após a execução.

## Decisões

Propostas G1 a G8 no desenho; viram ADRs 0016 a 0019 quando aprovadas.

## Problemas encontrados

Nenhum até agora.

## Pendências

- Aprovação das decisões G1 a G8.
- Autorização para criar os grupos no workspace (6.2).

## Aprendizados

A preencher ao longo da fase.

## Encerramento

A preencher.
