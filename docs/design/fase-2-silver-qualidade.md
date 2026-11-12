# Documento de Desenho Técnico — Fase 2: Silver e Qualidade

| | |
|---|---|
| **Projeto** | Olist Lakehouse |
| **Fase** | 2 — Silver e Qualidade |
| **Responsável técnico** | João Guilherme Araujo |
| **Data** | 2026-11-11 |
| **Status** | Aprovado em 2026-11-11 |
| **Documentos relacionados** | [Desenho da Fase 1](fase-1-simulador-bronze.md) · [ADR 0003 — Idempotência](../adr/0003-idempotencia-ponta-a-ponta.md) · [ADR 0004 — JSON inválido na bronze](../adr/0004-linhas-json-invalidas-na-bronze.md) · [Dicionário de métricas v0](../dicionario_metricas.md) |

---

## 1. Objetivo deste documento

Registrar o desenho técnico da camada silver **antes** da implementação: o que entra, como é tratado, onde ficam os registros inválidos e como saberemos que o resultado está correto. As decisões foram aprovadas em 2026-11-11.

## 2. Contexto e alinhamento com o negócio

Hoje a bronze guarda o dado exatamente como chegou, e por isso carrega todas as falhas da origem. Se áreas diferentes calcularem GMV ou % de entregas no prazo direto dela, cada uma chegará a um número diferente. A silver é a camada em que o dado vira **confiável e único**: tipado, sem duplicatas, ordenado pelo tempo do negócio e com os registros inválidos separados e explicados.

**Requisitos de negócio:**
- Um pedido, item, pagamento ou avaliação aparece uma só vez na silver.
- Nenhum registro inválido some em silêncio: ou é corrigido por uma regra documentada, ou vai para a quarentena com o motivo.
- É possível mostrar, por tabela e por regra, quantos registros passaram, foram corrigidos e foram para a quarentena.
- Os números da silver reproduzem o gabarito do simulador (`ops.simulator_chaos_log`).

## 3. Estado de partida (bronze em 2026-11-11)

Simulação de 2026-09-06 a 2028-08-21, 27 execuções, com 24.726 falhas no gabarito. Todas as colunas de negócio da bronze são `string` (o schema inferido não converte tipos).

| Tabela bronze | Linhas | Particularidades observadas |
|---|---|---|
| `orders` | 106.575 (98.476 pedidos distintos) | reenvio de arquivo inteiro (`_retry-1.json`) repete o dia todo |
| `order_items` | 117.080 | `price` com formato fora do padrão (4 casos), 1 linha com `_corrupt_record`, chegada atrasada (3.381) |
| `order_payments` | 107.928 | `payment_gateway` nula antes da coluna surgir (2026-10-08) |
| `order_reviews` | 101.365 | — |
| `order_status_events` | 408.678 | chegada atrasada (11.565) e fora de ordem (655); chave natural: `order_id` + `event_sequence` |
| `customers` / `sellers` | 106.667 / 3.430 | feeds de mudança (`operation` = INSERT/UPDATE/DELETE, `change_ts`, `change_sequence`) |
| `products`, `geolocation`, `category_translation` | 32.951 / 1.000.163 / 71 | carga única de referência |

## 4. Escopo

**Dentro do escopo:** pipeline de silver para as 10 entidades; tipagem; deduplicação; ordenação pelo tempo do negócio; expectations; quarentena e resultados de qualidade em `olist.quality`; validação contra o gabarito; documentação e tags das tabelas.

**Fora do escopo (fases seguintes):** aplicar o feed de `customers`/`sellers` como SCD1/SCD2 (Fase 3); regras de negócio das métricas, como "pedido válido", e o modelo dimensional (Fase 4); mascaramento e exclusão LGPD (Fase 6).

## 5. Visão geral da solução

```
olist.bronze.*  (Delta, tudo string, com duplicatas e falhas)
      │  leitura incremental (streaming table, trigger availableNow)
      ▼
Pipeline Lakeflow Spark Declarative Pipelines (1 pipeline, execução disparada)
      │  1. tipagem e normalização (funções puras DataFrame → DataFrame)
      │  2. expectations (regras nomeadas por tabela)
      ├──► registros válidos ──► deduplicação por chave (AUTO CDC, SCD tipo 1) ──► olist.silver.<entidade>
      └──► registros reprovados ─────────────────────────────────────────────► olist.quality.<entidade>_quarentena
                                                                                  (+ motivo da regra violada)
Log de eventos do pipeline ──► olist.quality.dq_resultados (passou / reprovou por regra e execução)
Validação: notebook compara silver + quarentena com a landing e com o gabarito
```

## 6. Perguntas e respostas de desenho

#### S1. Tecnologia e linguagem do pipeline
**Proposta:** Lakeflow Spark Declarative Pipelines em **Python**, com as transformações como funções puras em módulo próprio (testáveis sem cluster) e o arquivo do pipeline só declarando tabelas e regras.
**Por quê:** é o padrão de mercado da plataforma, dá checkpoint, incrementalidade e métricas de qualidade prontas, e a convenção do projeto exige funções puras.
**Alternativa:** SQL declarativo. Mais curto, mas as regras de tipagem ficam repetidas e difíceis de testar.
**Restrição do ambiente:** 1 pipeline ativo por tipo; execução disparada, não contínua.

#### S2. Quais entidades passam por qual tratamento
- **Transacionais** (`orders`, `order_items`, `order_payments`, `order_reviews`, `order_status_events`): tipagem, expectations, deduplicação e quarentena.
- **Referência** (`products`, `geolocation`, `category_translation`): tipagem e expectations; sem deduplicação por evento.
- **Cadastros com feed** (`customers`, `sellers`): nesta fase ficam apenas **tipados, validados e deduplicados por `change_sequence`**, mantendo as colunas `operation` e `change_ts`. A aplicação como SCD1/SCD2 é da Fase 3.

#### S3. Deduplicação e ordem
- Chave por entidade: `order_id`; `order_id` + `order_item_id`; `order_id` + `payment_sequential`; `review_id` + `order_id`; `order_id` + `event_sequence`.
- Duplicatas do tipo `duplicate_record` (mesma chave no mesmo arquivo) e `file_resend` (arquivo `..._retry-1.json`) são idênticas ao original: mantém-se uma linha por chave.
- Quando há mais de uma versão da mesma chave, vale a de maior `event_ts` e `event_sequence` — **nunca** o `sim_date`, que é o dia do arquivo e não o do negócio.
- Mecanismo: `create_auto_cdc_flow` com `stored_as_scd_type = 1`. É idempotente por construção e trata chegada atrasada e fora de ordem pelo `sequence_by`. Alternativa avaliada: `dropDuplicatesWithinWatermark`, que descarta duplicatas só dentro de uma janela de tempo e deixaria passar reenvios tardios.

#### S4. Tipagem
Conversão de `string` para o tipo de negócio com funções de conversão que devolvem nulo em vez de falhar (`try_cast`): decimais para `price`, `freight_value`, `payment_value`; inteiros para sequências e quantidades; timestamps para datas; `review_score` como inteiro de 1 a 5. Uma conversão que resulta em nulo onde havia valor é reprovada por expectation, e não descartada em silêncio.

#### S5. Expectations e quarentena
- Cada regra tem **nome**, e o resultado por regra é guardado.
- `expect_or_fail` só para violação que invalida o pipeline inteiro (por exemplo, schema da tabela destino diferente do contrato).
- `expect_or_drop` para registros inválidos, **combinado com uma tabela de quarentena** que recebe os mesmos reprovados com a coluna `motivo`. O `drop` sozinho faz o registro desaparecer sem rastro.
- `expect` (apenas mede) para regras de aviso, como `payment_gateway` nula, que é opcional no contrato.
- Regras mínimas: chave não nula; `_corrupt_record` nulo; `_rescued_data` nulo; valores numéricos conversíveis e não negativos; `review_score` entre 1 e 5; status dentro do domínio conhecido.

#### S6. Valores fora do padrão (`price` com formato diferente)
**Decisão (D1), aprovada com o desenho em 2026-11-11.**
- **Recomendação:** quando a correção é determinística e documentada (por exemplo, vírgula decimal), o registro é **corrigido na silver e marcado** em uma coluna de auditoria (`_dq_correcao`), com a contagem no relatório. Para o gabarito, `bad_value` fica como "corrigido", não como "perdido".
- **Alternativa:** enviar tudo para a quarentena. Mais conservador, mas retira vendas reais do GMV por um problema de formato.

#### S7. Colunas de auditoria e documentação
Todas as tabelas silver mantêm `_ingested_at`, `_source_file` e `_run_id` da bronze, mais o `_run_id` do pipeline. Cada tabela e coluna tem comentário; colunas com dado pessoal ficam tagueadas (`customer_unique_id`, CEP, cidade) para a Fase 6.

#### S8. Parametrização e ambientes
Catálogo e schema de destino vêm da configuração do pipeline, sem caminho fixo nos arquivos, para permitir o prefixo `dev_` na Fase 7.

#### S9. Como validar contra o gabarito
Um notebook de validação confere, por entidade:
1. linhas da bronze − duplicatas removidas = linhas da silver + linhas da quarentena;
2. as duplicatas removidas coincidem com o que se calcula de forma independente a partir das chaves na landing;
3. cada falha do gabarito (`corrupt_file`, `bad_value`, `new_column`, `late_arrival`, `out_of_order`) tem o destino esperado: quarentena, correção, coluna opcional ou ordem correta;
4. rodar o pipeline duas vezes não altera nenhuma contagem (idempotência, ADR 0003).
O `file_resend` do gabarito conta arquivos, e não registros; a comparação usa as linhas efetivamente repetidas.

## 7. Resumo das decisões

| # | Decisão | Status |
|---|---|---|
| S1 | Pipeline declarativo em Python, funções puras em módulo próprio | aprovada |
| S2 | `customers`/`sellers` só tipados e deduplicados; SCD na Fase 3 | aprovada |
| S3 | Dedup por AUTO CDC SCD1, ordem por `event_ts` + `event_sequence` | aprovada |
| S4 | Tipagem com conversão tolerante e expectation sobre nulos | aprovada |
| S5 | `expect_or_drop` + tabela de quarentena com motivo | aprovada |
| S6 | Valor fora do padrão: corrigir e marcar (D1) | aprovada |
| S7–S9 | Auditoria, parametrização e validação contra o gabarito | aprovada |

## 8. Riscos e pontos de atenção

| Risco | Mitigação |
|---|---|
| Cota do ambiente | execução disparada e incremental; evitar full refresh sem necessidade; acompanhar o tempo por execução |
| Primeira execução processa todo o histórico (cerca de 1,9 milhão de linhas) | rodar uma vez como carga inicial e medir antes de criar rotinas repetidas |
| Importar o módulo de funções no pipeline | verificar na primeira etapa como o caminho do módulo é resolvido pelo pipeline |
| `payment_gateway` nula em registros antigos | coluna opcional no contrato, só medida |

## 9. Plano de implementação e critérios de aceite

| Etapa | Entrega | Critério de aceite |
|---|---|---|
| 2.1 | Pipeline mínimo com `orders`: tipagem, dedup e quarentena ponta a ponta | 106.575 linhas da bronze viram 98.476 pedidos; rodar duas vezes não muda a contagem |
| 2.2 | Demais entidades transacionais e de referência | contagens conferem com as chaves distintas da landing |
| 2.3 | Expectations nomeadas, quarentena e valores fora do padrão | `_corrupt_record` e `bad_value` com o destino definido em S6 |
| 2.4 | `quality.dq_resultados` e notebook de validação contra o gabarito | todos os itens de S9 passando |
| 2.5 | `customers`/`sellers` tipados e deduplicados; ADR da fase; documentação e tags; encerramento | definição de pronto da fase atendida |
