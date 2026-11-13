# Fase 3 — CDC e SCD2

- **Período:** 2026-11-12 a 2026-11-13
- **Status:** ✅ concluída em 2026-11-13 — [desenho técnico](../design/fase-3-cdc-scd2.md) (decisões D1 a D6) e [ADR 0006](../adr/0006-historico-scd2-do-cadastro.md)
- **Branch/PRs:** `fase-3/scd2`, PR único (etapas 3.1 a 3.3)

## Objetivo

**Negócio:** saber o endereço e o cadastro de clientes e vendedores **na data de cada pedido**, e não só o atual, para que as métricas históricas não mudem quando alguém muda de cidade ou é excluído.

**Técnico:** dimensões com histórico (SCD tipo 2) construídas a partir do feed de mudanças (`operation`, `change_sequence`, `change_ts`) de `customers` e `sellers`, com `create_auto_cdc_flow` e tratamento explícito das exclusões.

Contexto de negócio, escopo e critérios de aceite estão no [desenho técnico](../design/fase-3-cdc-scd2.md).

## Pendências herdadas da Fase 2

Ver "Encerramento" em [fase-2-silver-qualidade.md](fase-2-silver-qualidade.md), em especial:

- a silver SCD1 só guarda a última versão; o histórico vem do feed;
- 288 clientes e 21 vendedores têm `DELETE` como última versão, e a linha `DELETE` chega sem atributos;
- `change_ts` e `sim_date` do cadastro chegam a 2028.

## Etapas

Plano da seção 9 do [desenho técnico](../design/fase-3-cdc-scd2.md):

| Etapa | Entrega | Status |
|---|---|---|
| 3.1 | SCD2 de `sellers` e a consulta "como era na data X" | ✅ executada e conferida (3.361 versões · 3.095 chaves · 3.074 abertas · 0 excluídos com versão aberta) |
| 3.2 | SCD2 de `customers` | ✅ executada e conferida (101.008 versões · 98.476 chaves · 98.188 abertas; 98.476 de 98.476 pedidos acham uma versão na data da compra) |
| 3.3 | Verificações 13 a 17, documentação, ADR e encerramento | ✅ validação `9e0b20e8` (2026-11-13 17:37): **71 OK, 0 reprovadas**; comentários e tags aplicados |

## O que foi feito

- **2026-11-12 — abertura da fase:** PR #9 (Fase 2) integrado à `main`. Levantamento dos dados dos feeds e rascunho do desenho. Fatos que definem o desenho: cada chave tem no máximo 3 versões (`change_sequence` 1, 2 ou 3); todo UPDATE é uma mudança de endereço (266 de 266 vendedores mudaram o CEP); `change_ts` cresce com a versão em todas as chaves; dos 266 vendedores que se mudaram, 265 têm itens vendidos antes (3.906) e depois (7.347) da mudança; os 2.532 clientes com UPDATE mudaram depois do pedido; as linhas `DELETE` chegam sem atributos. Versões esperadas no histórico: 101.008 (`customers`) e 3.361 (`sellers`).
- **2026-11-12/13 — etapas 3.1 e 3.2:** `02_silver_historico` adicionado ao pipeline `olist.silver`; `sellers_historico` 3.361 versões · 3.095 chaves · 3.074 abertas · 0 excluídos com versão aberta; `customers_historico` 101.008 · 98.476 · 98.188; 98.476 de 98.476 pedidos acham uma versão na data da compra; intervalos encadeados sem sobreposição; versão aberta igual à silver SCD1; 2.526 pedidos mostram CEP diferente do vigente na compra (os demais dos 2.532 clientes com histórico foram excluídos depois).
- **2026-11-13 — etapa 3.3 preparada:** verificações 13 a 17 acrescentadas ao `02_validacao_silver`, comentários e tags de PII nas duas tabelas de histórico (`03_documentacao_pii`) e [ADR 0006](../adr/0006-historico-scd2-do-cadastro.md).
- **2026-11-13 — etapa 3.3 concluída:** validação `9e0b20e8` (17:37): 95 linhas, **71 verificações OK, 0 reprovadas** (as 55 da Fase 2 mais 16 novas: 8 por tabela de histórico). `customers_historico`: 101.008 versões, 98.188 abertas, 288 exclusões fechadas na data do `DELETE`, 101.008 de 101.008 versões reproduzidas pela consulta por data. `sellers_historico`: 3.361 versões, 3.074 abertas, 21 exclusões fechadas, 3.361 de 3.361 reproduzidas. Comentários em 15/15 e 14/14 colunas; tags de coluna (`pii` e `pii_categoria`: 4 em `customers_historico`, 3 em `sellers_historico`) e `contem_pii` nas duas tabelas. Uma execução normal do pipeline depois da validação (17:38) manteve as contagens (101.008 · 3.361 · 98.476 · 3.095) e as tags.

## Decisões

Todas consolidadas no [ADR 0006](../adr/0006-historico-scd2-do-cadastro.md).

- **2026-11-12:** desenho aprovado sem alterações: D1 (manter SCD1 e acrescentar `_historico`), D2 (`DELETE` fecha a versão), D3 (`sequence_by = change_ts`), D4 (versão só por atributos de negócio), D5 (mesmo pipeline, segundo arquivo-fonte), D6 (verificações 13 a 17 no `02_validacao_silver`). Na implementação, `change_ts` também ficou fora do rastreamento (é a coluna de ordenação, não um atributo do cadastro).

## Problemas encontrados e como foram resolvidos

| Problema | Causa | Solução |
|---|---|---|
| Nenhum erro de execução nas três etapas | — | O desenho foi fechado olhando os dados antes (versões por chave, `change_ts` crescente, `DELETE` sem atributos), e o roteiro de cada etapa trazia as consultas de conferência com o valor esperado. |

## Aprendizados

- `__START_AT` e `__END_AT` herdam o tipo da coluna de ordenação: escolher `change_ts` e não `change_sequence` é o que torna o histórico consultável por data.
- A linha `DELETE` no SCD2 só fecha a versão (`apply_as_deletes`); o `__END_AT` já diz até quando o cadastro existiu.
- O que não é atributo de negócio (auditoria, metadados do feed) precisa ficar fora de `track_history_except_column_list`, senão um reenvio abre versão falsa.
- Em SCD2, o intervalo é fechado no início e aberto no fim (`>=` e `<`): cada instante pertence a uma versão só.
- A função `registrar_historico` escrita por dados fez `customers` entrar com uma linha, como na Fase 2.
- Consultar a tabela por outro caminho (CLI/SQL) antes de entregar o roteiro economizou uma volta: os valores esperados já estavam conferidos.
- A idempotência do histórico foi conferida por contagem após uma execução normal sem dados novos; a segunda validação formal (verificação 10 do histórico) fica como rotina para a Fase 7.

## Encerramento — pendências para as próximas fases

- **Fase 4 (gold):** ligar pedido e item à versão do cadastro **na data do pedido** com junção por intervalo (`>= __START_AT` e `< __END_AT`), documentar a convenção da fronteira e decidir o tratamento dos 288 pedidos de clientes excluídos e dos itens dos 21 vendedores excluídos (a versão existe, mas está fechada); `dim_customer` e `dim_seller` nascem destas tabelas.
- **Fase 6:** o histórico guarda os **endereços antigos**, então a exclusão de dados pessoais precisa alcançar `customers_historico` (e o `VACUUM`); política de acesso por categoria de PII; evidência auditável.
- **Fase 7:** testes unitários de `registrar_historico` e das listas de colunas; módulo único para funções e chaves (hoje repetidas entre notebooks); orquestração do pipeline e da validação em job; segunda validação automática para provar idempotência do histórico.
- **Pendências herdadas:** proteção da `main` no GitHub; últimos 2 meses do dataset para a Fase 5.