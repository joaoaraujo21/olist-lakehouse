# Fase 6 — Governança e LGPD

- **Período:** a iniciar
- **Status:** ⚪ a iniciar — desenho técnico a escrever antes da implementação
- **Branch/PRs:** a definir (uma branch e um PR por etapa)

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

## Etapas (previstas)

| Etapa | Entrega | Status |
|---|---|---|
| 6.1 | Desenho técnico e decisões (perfis, máscaras, exclusão) | a iniciar |
| 6.2 | Grants por perfil e máscaras de coluna | a iniciar |
| 6.3 | Filtros de linha | a iniciar |
| 6.4 | Exclusão de cliente ponta a ponta com evidência | a iniciar |
| 6.5 | Documentação, ADRs e encerramento | a iniciar |

## O que foi feito

Nada ainda.

## Decisões

A registrar em ADR.

## Problemas encontrados

Nenhum até agora.

## Aprendizados

A preencher ao longo da fase.

## Encerramento

A preencher.
