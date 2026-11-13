# Documento de Desenho Técnico — Fase 3: CDC e SCD2

| | |
|---|---|
| **Projeto** | Olist Lakehouse |
| **Fase** | 3 — CDC e SCD2 |
| **Responsável técnico** | João Guilherme Araujo |
| **Data** | 2026-11-12 |
| **Status** | Aprovado em 2026-11-12 (D1 a D6) · implementado e validado em 2026-11-13 |
| **Documentos relacionados** | [Desenho da Fase 2](fase-2-silver-qualidade.md) · [ADR 0005 — Silver declarativa](../adr/0005-silver-declarativa-deduplicacao-e-quarentena.md) · [Dicionário de métricas v0](../dicionario_metricas.md) |

---

## 1. Objetivo deste documento

Registrar o desenho do histórico de cadastro **antes** da implementação: o que é versionado, como a exclusão é tratada, como a consulta "como era na data X" funciona e como saberemos que o histórico está correto.

## 2. Contexto e alinhamento com o negócio

Vendedores mudam de cidade e clientes mudam de endereço. A silver da Fase 2 guarda só a **última** versão de cada cadastro (SCD tipo 1), então toda análise passa a enxergar o endereço de hoje, mesmo para um pedido de meses atrás. Isso faz a análise histórica mentir: uma venda feita por um vendedor de Campinas passa a aparecer como venda de Curitiba quando ele se muda.

**Requisitos de negócio** (épico E4 do escopo):
- Responder "como era o vendedor X em determinada data" (cidade, UF e CEP naquela data).
- Atribuir cada venda ao cadastro vigente **na data da venda**, e não ao cadastro de hoje.
- Tratar a exclusão de um cadastro sem perder a prova de que ele existiu, e sem deixar pedidos antigos sem dono.
- O resultado precisa ser reproduzível a partir da landing e conferível contra os arquivos originais.

## 3. Estado de partida (dados em 2026-11-12)

Feeds de `customers` e `sellers` na bronze, já tipados e deduplicados em `silver.customers` e `silver.sellers` (uma linha por chave, a última versão, com `operation` e `change_ts`).

| Fato (consultado nos dados) | customers | sellers |
|---|---|---|
| Chaves distintas | 98.476 | 3.095 |
| Chaves só com INSERT (`change_sequence` máxima = 1) | 95.662 | 2.809 |
| Chaves com UPDATE como última versão (2) | 2.526 | 265 |
| Chaves com DELETE como última versão (3) | 288 | 21 |
| Versões distintas que têm atributos (INSERT + UPDATE) | 101.008 | 3.361 |
| Todo UPDATE é uma **mudança de endereço**? | — | sim: 266 de 266 mudaram o CEP; 251 a cidade; 165 a UF |
| `change_ts` cresce com `change_sequence` em todas as chaves? | sim (2.826 pares, 0 violações) | sim (288 pares, 0 violações) |
| Falhas do gabarito | `duplicate_record` 1.018, `file_resend` 25 | `duplicate_record` 34, `file_resend` 11 |
| Chegada atrasada / fora de ordem no CDC? | não foi injetada | não foi injetada |

Mais fatos que pesam nas decisões:
- Cada chave tem no máximo **3 versões** (INSERT, UPDATE, DELETE): o `change_sequence` vale 1, 2 ou 3.
- A linha `DELETE` chega **sem atributos** (cidade, UF, CEP e `customer_unique_id` nulos).
- Dos 266 vendedores que se mudaram, 265 têm itens vendidos: 3.906 **antes** da mudança (101 vendedores) e 7.347 **depois** (213 vendedores). É exatamente o caso em que o cadastro atual atribuiria vendas ao endereço errado.
- Os 2.532 clientes com UPDATE mudaram **depois** do pedido (todos os 2.532): o endereço de entrega do pedido é o da versão original.
- 288 pedidos pertencem a clientes excluídos; 280 itens pertencem aos 21 vendedores excluídos, todos vendidos **antes** da exclusão.
- Todos os INSERTs de vendedores têm `change_ts = 2026-09-06 00:00:00` (carga inicial): antes dessa data não há cadastro de vendedor.

## 4. Escopo

**Dentro do escopo:** histórico SCD2 de `customers` e `sellers` a partir do feed; tratamento da exclusão; consulta "como era na data X"; validação do histórico contra a landing; documentação e tags.

**Fora do escopo (fases seguintes):** dimensões da gold (`dim_customer`, `dim_seller`) e junção das vendas ao cadastro vigente (Fase 4); mascaramento, exclusão auditável e `VACUUM` (Fase 6); testes unitários e CI (Fase 7).

## 5. Visão geral da solução

```
bronze.customers / sellers
      │  (já existe, Fase 2) customers_tipado → customers_validos (view com expectations)
      ├──► AUTO CDC SCD1  ──► silver.customers / silver.sellers       (estado atual; já entregue)
      └──► AUTO CDC SCD2  ──► silver.customers_historico / sellers_historico   (uma linha por versão)
                               colunas __START_AT e __END_AT (intervalo de validade)
Consulta "como era na data X":  WHERE data >= __START_AT AND (data < __END_AT OR __END_AT IS NULL)
Validação: histórico × landing × SCD1 (nova seção do 02_validacao_silver)
```

## 6. Perguntas e respostas de desenho

#### D1. Substituir a silver SCD1 ou manter e acrescentar o histórico?
**Proposta:** **manter** `silver.customers` e `silver.sellers` (estado atual, já validada) e **acrescentar** `silver.<e>_historico` (SCD2), as duas alimentadas pela mesma view validada.
**Por quê:** o custo é irrisório (98 mil e 3 mil linhas), a Fase 2 não precisa ser reaberta, e as duas tabelas permitem uma verificação cruzada: a versão aberta do histórico tem que ser igual à linha do SCD1 (exceto exclusões).
**Alternativa:** só o histórico, com o estado atual como `WHERE __END_AT IS NULL`. Menos tabelas, mas obriga a refazer a validação da Fase 2 e perde a verificação cruzada.

#### D2. O que fazer com o `DELETE` no histórico?
**Proposta:** `apply_as_deletes` no AUTO CDC SCD2: o `DELETE` **fecha** a versão vigente (preenche `__END_AT` com a data da exclusão) e não cria linha nova. O histórico preserva as versões anteriores, então fica provado que o cadastro existiu e até quando.
**Por quê:** a linha `DELETE` vem sem atributos; guardá-la como versão poluiria o histórico com uma linha toda nula. Fechar o intervalo diz tudo que importa: "existiu de X até Y".
**Alternativa:** guardar o `DELETE` como uma versão com `operation = 'DELETE'` e atributos nulos. Mantém o evento, mas obriga todo consumidor a filtrá-lo.
**Fora desta fase:** apagar os dados pessoais do histórico (direito ao esquecimento) é da Fase 6, com evidência.

#### D3. Qual a coluna de ordenação (`sequence_by`) do SCD2?
**Proposta:** `change_ts`, e não `change_sequence`.
**Por quê:** `__START_AT` e `__END_AT` têm o mesmo tipo da coluna de ordenação (documentação do AUTO CDC). Com `change_sequence`, o intervalo seria "de 1 até 2", inútil para "como era na data X". Com `change_ts`, o intervalo é uma data. Os dados mostram que `change_ts` cresce com a versão em todas as chaves, então não há ambiguidade, e duplicatas têm o mesmo `change_ts`.
**Observação:** na silver SCD1 da Fase 2, `change_sequence` continua sendo a ordenação; aqui o tempo importa. Como a unicidade por (chave, `change_ts`) é uma premissa, a validação a confere.
**Alternativa:** `struct(change_ts, change_sequence)`, que desempata por versão. Mais robusta contra empate, mas `__START_AT` passaria a ser um struct, mais difícil de consultar.

#### D4. Quais colunas contam como mudança de versão?
**Proposta:** só os atributos de negócio, com `track_history_except_column_list` excluindo `operation`, `change_sequence`, `sim_date` e as colunas de auditoria (`_ingested_at`, `_source_file`, `_source_file_modified_at`, `_run_id`).
**Por quê:** sem a exclusão, uma duplicata reenviada, com `_ingested_at` diferente, criaria uma versão falsa. Só cidade, UF e CEP (e `customer_unique_id`) devem abrir versão.

#### D5. Onde mora o código do SCD2?
**Proposta:** no **mesmo pipeline** `olist.silver` (o ambiente permite um pipeline ativo por tipo), em um **segundo notebook-fonte**, `src/silver/02_silver_historico`, que só declara as duas tabelas e os dois fluxos lendo as views `customers_validos` e `sellers_validos`, que já existem no pipeline.
**Por quê:** as views de um pipeline são visíveis entre os arquivos-fonte; não precisa duplicar a tipagem nem as regras, e o notebook 01 fica intocado.
**Risco:** a execução em duas fontes precisa ser verificada no *Dry run* (ver 8).

#### D6. Onde fica a validação?
**Proposta:** nova seção no `02_validacao_silver` (verificações 13 a 17), com a mesma tabela de resultados.
**Por quê:** uma validação só, um lugar onde olhar; a idempotência e a falha no fim já existem.

## 7. Resumo das decisões

| # | Decisão | Status |
|---|---|---|
| D1 | Manter SCD1 e acrescentar `silver.<e>_historico` (SCD2) | aprovada |
| D2 | `DELETE` fecha a versão (`apply_as_deletes`), sem linha nova | aprovada |
| D3 | `sequence_by = change_ts` no SCD2 | aprovada |
| D4 | Versão só por atributos de negócio (`track_history_except_column_list`) | aprovada |
| D5 | Mesmo pipeline, segundo notebook-fonte `02_silver_historico` | aprovada |
| D6 | Validação acrescentada ao `02_validacao_silver` (13 a 17) | aprovada |

## 8. Riscos e pontos de atenção

- **Dois arquivos-fonte no mesmo pipeline:** é preciso adicionar o notebook novo à lista de fontes (lição da Fase 2: o pipeline não enxerga o que não está na lista). Conferir no *Dry run*.
- **Tabela SCD2 nova entra com execução normal**, sem *full refresh*; as tabelas existentes não são tocadas.
- **`__START_AT` e `__END_AT` herdam o tipo de `change_ts`** (`TIMESTAMP`); o `DELETE` define o `__END_AT` da versão.
- **Empate de `change_ts` na mesma chave:** nos dados atuais não ocorre; a verificação 14 o detecta se aparecer.
- **Datas até 2028:** `change_ts` e `sim_date` do cadastro vão até 2028-08-21 porque a simulação foi avançada; isso é consistente com as datas dos pedidos, e não um erro.
- **Pedidos de clientes excluídos (288):** a Fase 4 precisa de uma junção por intervalo de datas e de uma regra para o pedido cujo cliente foi encerrado.

## 9. Plano de implementação e critérios de aceite

| Etapa | Entrega |
|---|---|
| 3.1 | `02_silver_historico`: SCD2 de `sellers` (a entidade do épico E4) e a consulta "como era na data X" |
| 3.2 | SCD2 de `customers`, com a mesma função declarada por dados |
| 3.3 | Verificações 13 a 17 no `02_validacao_silver`, comentários e tags de PII nas tabelas novas, ADR 0006, documentação e encerramento |

**Critérios de aceite** (contagens esperadas, deduzidas dos dados):

| # | Verificação | Esperado |
|---|---|---|
| 13 | Linhas do histórico = versões distintas com atributos | `customers_historico` 101.008 · `sellers_historico` 3.361 |
| 14 | Intervalos sem sobreposição e sem empate por chave (`__END_AT` de uma versão = `__START_AT` da seguinte) | 0 violações |
| 15 | Versões abertas (`__END_AT` nulo) = chaves não excluídas; chaves excluídas não têm versão aberta | customers 98.188 · sellers 3.074 |
| 16 | Versão aberta = linha do SCD1 (exceto exclusões) | 0 divergências |
| 17 | Consulta "como era na data X" reproduz a landing: para cada vendedor que se mudou, antes da mudança o cadastro é o do INSERT e depois é o do UPDATE | 266 de 266 |
| — | Idempotência: executar de novo, sem dados novos, não muda nada | todas as contagens iguais |
