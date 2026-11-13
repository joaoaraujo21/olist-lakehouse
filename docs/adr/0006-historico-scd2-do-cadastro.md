# ADR 0006 — Histórico de cadastro em SCD tipo 2 ao lado do estado atual

- **Status:** aceito
- **Data:** 2026-11-13
- **Fase:** 3 — CDC e SCD2

## Contexto

Vendedores mudam de cidade e clientes mudam de endereço. A silver da Fase 2 guarda só a última versão de cada cadastro (SCD tipo 1), então uma venda antiga passa a aparecer com o endereço de hoje. Nos dados: dos 266 vendedores que se mudaram, 265 têm vendas antes e depois da mudança (3.906 itens antes, 7.347 depois), e os 2.532 clientes com UPDATE mudaram depois do pedido. O feed de mudanças (`operation`, `change_sequence`, `change_ts`) traz no máximo 3 versões por chave (INSERT, UPDATE, DELETE), e a linha `DELETE` chega sem atributos. O épico E4 exige responder "como era o vendedor X em determinada data".

## Decisões

**1. Manter a silver SCD1 e acrescentar `silver.<entidade>_historico` (SCD2).** As duas tabelas são alimentadas pela mesma view validada (`<entidade>_validos`), então regras de qualidade e quarentena não se duplicam. O custo é irrisório (98 mil e 3 mil chaves), a Fase 2 não é reaberta e a verificação cruzada fica possível: a versão aberta do histórico tem que ser igual à linha do SCD1.

**2. `create_auto_cdc_flow` com `stored_as_scd_type=2`, ordenado por `change_ts`.** `__START_AT` e `__END_AT` herdam o tipo da coluna de ordenação; com `change_sequence` o intervalo seria "de 1 até 2", inútil para consulta por data. `change_ts` cresce com a versão em todas as chaves, e duplicatas têm o mesmo valor. A premissa de que não há empate de `change_ts` na mesma chave é conferida pela validação.

**3. O `DELETE` fecha a versão vigente (`apply_as_deletes`), sem criar linha.** A linha `DELETE` vem sem atributos; guardá-la como versão poluiria o histórico com uma linha nula. O intervalo fechado diz o que importa: "existiu de X até Y", e os pedidos antigos continuam com dono. Apagar os dados pessoais do histórico (direito ao esquecimento) é decisão da Fase 6.

**4. Só atributos de negócio abrem versão (`track_history_except_column_list`).** Ficam de fora a auditoria (`_ingested_at`, `_source_file`, `_source_file_modified_at`, `_run_id`), os metadados do feed (`operation`, `change_sequence`, `change_ts`, `sim_date`) e as colunas técnicas da tipagem. Sem isso, uma duplicata reenviada, com auditoria diferente, abriria uma versão falsa.

**5. Mesmo pipeline `olist.silver`, segundo arquivo-fonte (`src/silver/02_silver_historico`).** O ambiente permite um pipeline ativo, e as views do pipeline são visíveis entre os arquivos-fonte; o `01_silver_pipeline` ficou intocado. A função `registrar_historico` é escrita uma vez; `customers` entrou acrescentando uma linha à lista.

**6. Intervalo fechado no início e aberto no fim** (`ts >= __START_AT AND (ts < __END_AT OR __END_AT IS NULL)`). O instante exato da mudança pertence só à versão nova; nenhuma data cai em duas versões.

**7. Provado fora do pipeline.** Cinco verificações no `02_validacao_silver` recalculam o histórico esperado a partir da landing (janelas e junções em PySpark): linhas = versões; intervalos encadeados sem sobreposição; versões abertas = chaves vigentes, exclusões fechadas na data do `DELETE`; versão aberta = SCD1; "como era na data X" reproduz cada versão da landing.

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Substituir a silver SCD1 pelo histórico (estado atual = `__END_AT IS NULL`) | obriga a refazer a validação da Fase 2 e perde a verificação cruzada entre as duas formas |
| `sequence_by = struct(change_ts, change_sequence)` | desempata empates, mas `__START_AT` viraria um struct, mais difícil de consultar; não há empates nos dados e a validação os detecta |
| Guardar o `DELETE` como versão com atributos nulos | todo consumidor teria que filtrá-la; o `__END_AT` já carrega a informação |
| Rastrear todas as colunas | duplicatas reenviadas abririam versões falsas |
| Construir o histórico com `MERGE` à mão | reimplementa ordem, empate e fechamento de intervalo que o AUTO CDC já faz |
| Pipeline separado para o histórico | só há um pipeline ativo no Free Edition, e a view validada já está no pipeline atual |

## Consequências

- O histórico contém **dado pessoal** (endereços antigos), então é a tabela mais sensível do cadastro; recebeu comentários e as mesmas tags de PII (`contem_pii`, `pii`, `pii_categoria`). A política de acesso e a exclusão auditável são da Fase 6.
- A Fase 4 liga pedido e item à versão vigente **na data do pedido** com uma junção por intervalo, e precisa decidir como tratar os 288 pedidos de clientes excluídos (a versão existe, mas está fechada).
- `change_ts` e `sim_date` do cadastro vão até 2028 porque a simulação foi avançada; é consistente com as datas dos pedidos.
- Regra nova que deva valer para o histórico exige *full refresh*; tabela nova entra com execução normal.
