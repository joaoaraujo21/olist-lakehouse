# ADR 0020 — Propagação da exclusão por reconstrução e purga física

- **Status:** aceito (implementação na etapa 6.4)
- **Data:** 2026-11-23
- **Fase:** 6 — Governança e LGPD
- **Complementa:** [ADR 0019](0019-exclusao-de-cliente-e-evidencia.md)

## Contexto

O ADR 0019 previa `DELETE` em cada camada. Ao detalhar a implementação, três fatos mudaram o caminho:

1. A silver lê a bronze em streaming. Um `DELETE` na bronze faz a próxima atualização falhar (mesmo problema do ADR 0015), e `silver.customers`, `silver.customers_historico` e `quality.customers_quarentena` são tabelas gerenciadas pelo pipeline, com cópias físicas `__materialization_*`.
2. O serverless não deixa desligar a verificação de segurança do `VACUUM` (a configuração de sessão não está disponível). A retenção mínima só baixa por propriedade da tabela (`delta.deletedFileRetentionDuration`), o que foi testado em tabela de teste.
3. O histórico da tabela (`DESCRIBE HISTORY`) grava o predicado do `DELETE`. Um predicado com o identificador em claro deixaria a pessoa no log de transações.

## Decisão

1. **Apagar na bronze e reconstruir o derivado.** O `DELETE` ocorre só em `bronze.customers`; `customers`, `customers_historico` e `customers_quarentena` da silver são reconstruídas por *full refresh seletivo* e a gold é atualizada. A pessoa some por reconstrução, sem mudar o código do pipeline.
2. **Alvos em tabela, não no predicado.** Os ids do titular vão para `ops.exclusoes_alvos` (transitória, esvaziada ao fim) e os `DELETE` usam subconsulta, para o identificador não aparecer em `DESCRIBE HISTORY`.
3. **Purga com retenção zero por tabela.** Na bronze e nas cópias físicas `__materialization_*` das tabelas do pipeline (as tabelas do pipeline em si não aceitam alterar propriedades, e seus arquivos ficam nas cópias): `REORG ... APPLY (PURGE)`, propriedade `delta.deletedFileRetentionDuration = 0 hours`, `VACUUM ... RETAIN 0 HOURS` e remoção da propriedade. Isso também apaga os arquivos do Change Data Feed e bloqueia o *time travel* para versões anteriores.
4. **Pedido anônimo e recompra.** O pedido permanece com `customer_id` e sem `customer_unique_id`. A pessoa sai do cálculo da taxa de recompra; é uma variação mínima e conhecida (uma pessoa em cerca de 96 mil), registrada no dicionário. A janela `pedidos_validos_do_cliente` passa a particionar por `coalesce(customer_unique_id, customer_id)`, para os pedidos anônimos não formarem um grupo único.
5. **Evidência.** `ops.exclusoes_lgpd` (hash do titular, contagens antes e depois, versões Delta), etapa `6.4-exclusao` em `quality.validacao_governanca` e ausência do identificador em `DESCRIBE HISTORY`.

## Alternativas consideradas

1. **`DELETE` em cada camada com `skipChangeCommits`.** Exige alterar as leituras do pipeline silver e tratar as tabelas de apoio do AUTO CDC; mais código e mais risco.
2. **Token anônimo no pedido para manter a recompra.** Preserva o KPI, mas o hash continua ligando os pedidos da pessoa e pode ser dado pessoal pela LGPD.
3. **`VACUUM` padrão de 7 dias.** Seguro, mas a pessoa continua nos arquivos e no *time travel* por uma semana; não atende ao pedido de exclusão.

## Consequências

- (+) Nenhuma mudança no código da silver; a exclusão vale para o histórico, a quarentena e o Change Data Feed.
- (+) O identificador não fica no log de transações.
- (−) O full refresh reprocessa três tabelas inteiras a cada exclusão; em produção, exclusões em lote.
- (−) Retenção zero apaga o *time travel* das tabelas tocadas; é o efeito desejado e é registrado.
- (−) **Fora do escopo:** os arquivos brutos da landing (`raw_data`) e o dataset original (`olist_source`) ainda contêm a pessoa. Em produção, o dado bruto segue a política de retenção da origem; aqui é risco aceito e documentado. O log de auditoria do Databricks (`system.access.audit`) guarda os comandos executados e é retido pela plataforma.
