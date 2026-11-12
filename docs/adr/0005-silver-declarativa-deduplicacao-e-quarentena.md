# ADR 0005 — Silver declarativa: deduplicação por chave, quarentena e correção de formato

- **Status:** aceito
- **Data:** 2026-11-12
- **Fase:** 2 — Silver e Qualidade

## Contexto

A bronze guarda o dado como chegou: tudo em `string`, com duplicatas (`duplicate_record`, `file_resend`), registros atrasados e fora de ordem, uma coluna que surgiu sem aviso (`payment_gateway`), um preço com vírgula decimal e uma linha de JSON inválido. O gabarito do simulador (`ops.simulator_chaos_log`) registra cada falha injetada. A silver precisa entregar uma linha por chave, com tipos corretos, separar o que reprova nas regras de qualidade e provar o resultado contra o gabarito, sem custo que estoure a cota do Free Edition.

## Decisões

**1. Pipeline declarativo (Lakeflow Spark Declarative Pipelines), uma função para todas as entidades.** Cada entidade é descrita por dados (`Entidade`: tipos, chaves, obrigatórias, `sequence_by`, regras, avisos, correções) e `registrar_entidade` declara o fluxo: view tipada → view com expectations → silver, mais a tabela de quarentena. Acrescentar uma entidade é acrescentar uma linha; `customers` e `sellers` entraram na etapa 2.5 sem mudar a função.

**2. Deduplicação com `create_auto_cdc_flow` SCD tipo 1, ordenado pelo tempo do negócio.** Mantém uma linha por chave e vence a versão de maior `sequence_by`, o que trata duplicata, reenvio, chegada atrasada e fora de ordem de uma vez só. É idempotente: sem dados novos, a silver não muda. O `sequence_by` nunca é o `sim_date`, com **uma exceção**: `order_payments` não tem nenhum timestamp de negócio e usa `sim_date`; isso é seguro porque as duplicatas de pagamento são cópias idênticas. Nos feeds de `customers` e `sellers`, o `sequence_by` é `change_sequence` (1 INSERT, 2 UPDATE, 3 DELETE): dentro de cada par (chave, versão) o conteúdo é idêntico, então não há empate a desempatar.

**3. Tipagem tolerante, com o texto original preservado.** `try_cast` converte sem derrubar o pipeline (o valor inválido vira nulo) e o texto de origem fica em `<coluna>_bruto`, para a quarentena mostrar o que chegou. Uma coluna só reprova quando *havia valor e a conversão virou nulo*: um nulo que já vinha nulo é válido, a menos que a coluna seja obrigatória. Dinheiro é `DECIMAL(12,2)`, nunca `DOUBLE`.

**4. Quarentena com motivo, em vez de descartar.** `expect_all_or_drop` retira o registro da silver e registra a contagem por regra no log de eventos; a tabela `quality.<entidade>_quarentena` guarda os mesmos registros com o nome das regras violadas em `motivo`. As duas pontas usam o mesmo dicionário de regras, então nunca divergem. Regras que só medem (como `payment_gateway_preenchido`) usam `expect_all` e não retiram nada.

**5. Valor fora do padrão com formato conhecido é corrigido e marcado, não quarentenado.** O preço `59,90` é uma venda real; mandá-lo para a quarentena tiraria receita do GMV por um problema de formato. A correção vale só para o padrão exato `^\d+,\d{1,2}$` e fica registrada em `_dq_correcao`. Qualquer outro formato continua reprovado.

**6. Feeds de mudança (`customers`, `sellers`): uma linha por chave, a última versão, mantendo `operation` e `change_ts`.** A linha `DELETE` fica na silver (com os atributos nulos, como o feed a envia). O histórico (SCD2) e o tratamento das exclusões ficam para a Fase 3, e a exclusão auditável para a Fase 6. A regra de cadastro completo só vale quando a operação não é `DELETE`. O CEP é `string` para preservar zeros à esquerda.

**7. Qualidade medida e provada fora do pipeline.** `quality.dq_resultados` guarda o que o pipeline mediu (cópia do log de eventos, por MERGE por `update_id` + entidade + regra). `quality.validacao_resultados` guarda a auditoria independente: o notebook relê a landing, recalcula o esperado e compara com a silver, a quarentena e o gabarito, e falha no final se algo reprovar. Uma regra mal escrita pode aprovar tudo e o pipeline não notaria; a validação notaria.

**8. Comentários e tags de PII aplicados por `ALTER TABLE` num notebook, depois do pipeline.** Testado: `ALTER TABLE` é aceito nas streaming tables e comentários e tags sobrevivem a execuções do pipeline e a um *full refresh*.

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| `dropDuplicates` / `MERGE` escrito à mão | reimplementa o que o AUTO CDC já faz (ordem, empate, fora de ordem), com mais código e mais risco de erro; o AUTO CDC é a forma declarativa recomendada |
| `CAST` direto | derruba o pipeline no primeiro valor inválido; `try_cast` isola o problema no registro |
| `DOUBLE` para valores monetários | acumula erro de arredondamento na soma (GMV) |
| Mandar `price` com vírgula para a quarentena | tira venda real da receita; corrigir e marcar mantém o dado e a rastreabilidade |
| `expect_or_drop` sozinho | o registro some sem rastro; a tabela de quarentena guarda o que foi descartado e por quê |
| `expect_or_fail` | uma linha ruim pararia o pipeline todo; a quarentena permite seguir |
| Aplicar o feed como SCD2 já na silver | histórico é o objetivo da Fase 3; misturar aqui adiaria a prova da deduplicação |
| Apagar a linha na exclusão (`apply_as_deletes`) | perde a prova de que o cliente existiu e deixa pedidos órfãos; a exclusão controlada, com evidência, é da Fase 6 |
| Funções puras em módulo Python separado (previsto no desenho S1) | o caminho de importação de módulo dentro de um pipeline ainda não foi verificado; as funções ficaram no próprio notebook, e a extração fica para a Fase 7, junto com os testes |

## Consequências

- A silver tem 10 entidades: 7 deduplicadas por chave com AUTO CDC (`orders`, `order_items`, `order_payments`, `order_reviews`, `order_status_events`, `customers`, `sellers`) e 3 de referência, só tipadas e validadas (`products`, `geolocation`, `category_translation`; `geolocation` nem tem chave única).
- A validação cobre as 5 falhas do gabarito e o balanço de linhas (bronze = silver + quarentena + duplicatas), e a idempotência passa em todas as entidades.
- As regras de domínio ficam em dados, mas a chave de cada entidade está repetida nos notebooks de pipeline e de validação (ponto a resolver na Fase 7, com módulo único e testes unitários de `corrigir_formato` e `montar_regras`).
- `change_ts` e `sim_date` do feed de cadastro chegam a 2028 (a carga histórica foi avançada além do presente); a Fase 3 deve ordenar a história pelo tempo do negócio, não pela data de processamento.
- A linha `DELETE` na silver SCD1 tem os atributos nulos: quem precisa do último endereço conhecido usa a bronze ou o SCD2 da Fase 3.

## Referências

- Lakeflow Spark Declarative Pipelines — expectations, AUTO CDC e log de eventos (documentação da Databricks).
- Desenho técnico da fase: `docs/design/fase-2-silver-qualidade.md` (S1 a S9, D1).
- ADR 0004 — `_corrupt_record` e `_rescued_data` na bronze, tratados aqui como regras de quarentena.
