# ADR 0019 — Exclusão de cliente: cadastro apagado, pedido anônimo e evidência

- **Status:** aceito (implementação na etapa 6.4)
- **Data:** 2026-11-18
- **Fase:** 6 — Governança e LGPD
- **Ajustado por:** [ADR 0020](0020-propagacao-da-exclusao-por-reconstrucao-e-purga-fisica.md) (propagação por reconstrução e purga física)

## Contexto

A exclusão de cliente precisa ser ponta a ponta e auditável. O pedido é registro financeiro e entra nos KPIs oficiais. O estado ao vivo ignora `DELETE` na bronze (ADR 0015). As tabelas usam *deletion vectors*: o `DELETE` marca a linha, mas o dado continua nos arquivos até a purga. Uma pessoa (`customer_unique_id`) tem vários `customer_id`: 99.291 `customer_id` para 95.695 pessoas.

## Decisão

1. **Apagar o cadastro e manter o pedido sem vínculo com a pessoa.** Os KPIs oficiais não mudam; a base legal de guarda do registro do pedido fica documentada.
2. **Procedimento explícito, em notebook**, com parâmetro `customer_unique_id` e dois modos: `simulacao` (só conta) e `executar`.
   1. localizar todos os `customer_id` da pessoa e registrar a solicitação em `ops.exclusoes_lgpd` com o **hash** do titular, nunca o dado em claro;
   2. `DELETE` físico em `bronze.customers`, `silver.customers`, `silver.customers_historico` (as versões do SCD2 são **removidas**, não só fechadas como no ADR 0006) e nas quarentenas;
   3. reprocessar a gold: o pedido permanece, sem vínculo com a pessoa;
   4. `REORG TABLE ... APPLY (PURGE)` e `VACUUM` nas tabelas tocadas, inclusive para os arquivos de mudança do Change Data Feed;
   5. verificação independente: titular com 0 linhas em todas as tabelas, KPIs oficiais iguais e *time travel* sem a pessoa.
3. **Estado ao vivo:** como o pedido permanece, `gold_orders_live` não muda (não guarda dado pessoal). Isso encerra a pendência do ADR 0015.
4. **Evidência:** `ops.exclusoes_lgpd` (solicitação, contagens por tabela, versões Delta antes e depois, resultado), `DESCRIBE HISTORY` das tabelas e `system.access.audit`. O relatório do DPO não contém dado pessoal em claro.
5. Idempotente: repetir a exclusão da mesma pessoa não dá erro e registra 0 linhas.

## Alternativas consideradas

1. **Apagar também os pedidos.** Exclusão total, mas muda pedidos criados, GMV e entregas no prazo e apaga registro com obrigação de guarda.
2. **Propagar por Change Data Feed da bronze à silver.** Exigiria ligar o feed na bronze e tratar `delete` em cada camada; o procedimento explícito dá mais evidência por camada.
3. **Reconstruir tudo a cada exclusão.** Custa a carga completa por solicitação.

## Consequências

- (+) A pessoa some de todas as camadas, inclusive dos arquivos físicos, sem alterar os números oficiais.
- (+) Cada exclusão deixa registro auditável sem guardar o dado pessoal.
- (−) `VACUUM` com retenção mínima exige desligar a verificação de segurança, só dentro do procedimento e com registro.
- (−) O pedido anônimo continua tendo `customer_id`, que sem o cadastro não leva a ninguém, mas segue como chave técnica.
