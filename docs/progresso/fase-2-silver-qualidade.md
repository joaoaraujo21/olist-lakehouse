# Fase 2 — Silver e Qualidade

- **Período previsto:** a definir na abertura da fase
- **Status:** ✅ concluída em 2026-11-12 (etapas 2.1 a 2.5) — [desenho técnico](../design/fase-2-silver-qualidade.md) aprovado em 2026-11-11
- **Branch/PRs:** `fase-2/silver`, um PR único com a fase inteira

## Objetivo

**Negócio:** os números das áreas só convergem se o dado de base for confiável: sem duplicatas, com tipos corretos e com os registros inválidos separados e explicados.

**Técnico:** camada silver com Lakeflow Spark Declarative Pipelines sobre a bronze: tipagem, deduplicação, tratamento de chegada atrasada e fora de ordem, expectations e quarentena em `olist.quality`, validados contra o gabarito do simulador (`ops.simulator_chaos_log`).

O contexto de negócio, o escopo e os critérios de aceite serão detalhados na abertura da fase, com o desenho técnico aprovado antes da implementação.

## Pendências herdadas da Fase 1

Ver "Encerramento" em [fase-1-simulador-bronze.md](fase-1-simulador-bronze.md).

## Etapas

| Etapa | Entrega | Status |
|---|---|---|
| 2.1 | Pipeline mínimo com `orders`: tipagem, dedup e quarentena ponta a ponta | ✅ concluída (2026-11-12) |
| 2.2 | Demais entidades transacionais e de referência | ✅ concluída (2026-11-12) |
| 2.3 | Expectations nomeadas, quarentena e valores fora do padrão | ✅ concluída (2026-11-12) |
| 2.4 | `quality.dq_resultados` e notebook de validação contra o gabarito | ✅ concluída (2026-11-12) |
| 2.5 | `customers`/`sellers` tipados e deduplicados; ADR da fase; documentação, tags e encerramento | ✅ concluída (2026-11-12) |

## O que foi feito

- **2026-11-11 — carga histórica:** simulação avançada de 2026-11-01 a 2028-08-21 em lotes de 30 dias (simulador + bronze, serverless), 27 execuções no total, todas com sucesso; cada lote levou de 12 a 15 minutos. Landing = bronze em todas as 10 tabelas (ex.: `orders` 106.575, `order_status_events` 408.678). Gabarito com 24.726 falhas. O feed de mudanças já traz exclusões (304 em `customers`, 22 em `sellers`), o que resolve a pendência da Fase 3. Os últimos dois meses do dataset ficam para a Fase 5.
- **2026-11-11 — rascunho do desenho técnico** em `docs/design/fase-2-silver-qualidade.md`. 
- **2026-11-11 — desenho aprovado**, incluindo D1: valores fora do padrão são corrigidos e marcados em `_dq_correcao`.
- **2026-11-12 — etapa 2.1 especificada:** notebook-fonte do pipeline `src/silver/01_silver_pipeline` (`orders`): view tipada, view com expectations nomeadas (`expect_all_or_drop`), tabela `quality.orders_quarentena` com motivo e `create_auto_cdc_flow` SCD1 por `order_id` ordenado por `order_purchase_timestamp`. Funções puras declaradas no próprio notebook; extração para módulo fica para depois de verificar como o pipeline resolve o caminho do módulo (risco do desenho).
- **2026-11-12 — etapa 2.1 concluída:** pipeline `olist.silver` (serverless, disparado) executado: `silver.orders` com 98.476 linhas (todas com `order_id` distinto, a partir de 106.575 da bronze) e `quality.orders_quarentena` com 0 (o gabarito de `orders` só tem `duplicate_record` e `file_resend`). Segunda execução sem dados novos: contagem inalterada (idempotência). Duração: cerca de 30 s. Os dois riscos (expectation em view e AUTO CDC lendo view) não se confirmaram.
- **2026-11-12 — etapa 2.2 especificada:** o notebook passou a descrever cada entidade por dados (`Entidade`: tipos, chave, obrigatórias, `sequence_by`) e a registrar o fluxo inteiro em `registrar_entidade`, para as 8 entidades. Regras geradas por `montar_regras`; conversão que vira nulo onde havia valor é reprovada. Contagens esperadas na silver: `order_items` 111.588 (4 `bad_value` na quarentena até a 2.3), `order_payments` 102.906, `order_reviews` 95.253, `order_status_events` 387.691, `products` 32.951, `geolocation` 1.000.163, `category_translation` 71.
- **2026-11-12 — etapa 2.2 concluída:** execuções de 12:55 (carga inicial) e 12:58 (sem dados novos), ambas `COMPLETED`; contagens da silver idênticas às esperadas nas 8 entidades e sem duplicata de chave em `order_items`, `order_payments`, `order_reviews` e `order_status_events`. Quarentena: só `order_items` com linhas (4 `price` com vírgula decimal, motivo `price_nao_nulo, price_conversivel`, e 1 linha corrompida sem chave); as demais vazias, como no gabarito. Tipos conferidos (`price`/`freight_value` DECIMAL(12,2), `order_item_id` INT, datas TIMESTAMP, `sim_date` DATE).

- **2026-11-12 — etapa 2.3 especificada:** `regras_extras` (domínio), `avisos` (`expect_all`, só medem) e `correcoes` (`corrigir_formato`, `_dq_correcao`) em `Entidade`. Domínios conferidos no dado: 5 notas (1 a 5), 8 status, 4 tipos de pagamento, sem valores negativos; `payment_gateway` nulo em 133 pagamentos. Execução com *Full refresh all*, porque o streaming não relê os 4 itens de `price` que já estão na quarentena.

- **2026-11-12 — 2.3, primeira execução (full refresh, 13:27):** contagens corretas (`order_items` 111.592, demais iguais), quarentena de `order_items` só com a linha corrompida e as outras vazias, preços corrigidos (59.90, 30.97, 24.99, 77.90). Mas `_dq_correcao` ficou nula nos 4 itens (ver "Problemas"); correção feita no notebook, aguardando novo full refresh.

- **2026-11-12 — etapa 2.3 concluída:** após a correção de `corrigir_formato` (marca calculada no mesmo `withColumns`), full refresh (13:36) e execução normal (13:38) `COMPLETED`: `order_items` 111.592 linhas (4 com `_dq_correcao = price_virgula_decimal`: 59.90, 30.97, 24.99, 77.90), quarentena de `order_items` só com a linha corrompida, demais quarentenas vazias, demais silvers inalteradas. O log de eventos traz as contagens por regra (ex.: `payment_gateway_preenchido` 137 reprovados em 107.928 lidos, só medido; `price_conversivel` 4 reprovados na execução anterior à correção).
- **2026-11-12 — etapa 2.4 especificada:** dois notebooks regulares em `src/quality/`: `01_dq_resultados` (log de eventos → `quality.dq_resultados`, MERGE por `update_id`+`dataset`+`regra`) e `02_validacao_silver` (verificações 1 a 10 do desenho S9, resultado em `quality.validacao_resultados`, falha no fim se alguma reprovar). Observação para a Fase 4: 1.370 eventos em `order_status_events` têm `event_ts` anterior ao evento de `event_sequence` menor da mesma ordem; por isso a verificação de fora de ordem compara com a bronze, e não exige timestamps crescentes.

- **2026-11-12 — etapa 2.4, execução dos dois notebooks (14:08):** `quality.dq_resultados` com 249 linhas (8 entidades, 4 execuções com métricas), sem chave repetida. Validação `cccd4291`: 57 linhas, **33 verificações OK, 0 reprovadas, 24 sem comparação** (16 fotos de contagem + 8 de idempotência, por ser a primeira validação). Aprovadas: bronze = landing nas 8 entidades; balanço de linhas (ex.: `order_items` 117.080 = 111.592 silver + 1 quarentena + 5.487 duplicatas); silver = chaves distintas da landing e sem chave repetida nas 5 transacionais; `corrupt_file` 1 = 1; `bad_value` 4 = 4 (sem correção além do gabarito); `new_column` sem nulos após 2026-10-08; 3.381 itens e 11.565 eventos atrasados presentes; `out_of_order` sem divergência (387.691 eventos). Falta a prova da idempotência (verificação 10): rodar o pipeline sem dados novos e validar de novo.

- **2026-11-12 — etapa 2.4 concluída (idempotência):** execução normal do pipeline às 14:17 (`COMPLETED`, sem dados novos) e segunda validação `5bc7a7a0` (14:20): 57 linhas, **41 OK, 0 reprovadas**, 16 fotos de contagem. As 8 verificações de idempotência passaram a `ok = true`, com a silver igual à da validação anterior em todas as entidades (ex.: `orders` 98.476, `order_items` 111.592, `geolocation` 1.000.163). Todos os itens de S9 atendidos.
- **2026-11-12 — etapa 2.5 especificada:** `customers` e `sellers` acrescentados a `ENTIDADES` (duas linhas, sem mudar `registrar_entidade`), com `sequence_by = change_sequence`. Perfil dos dados antes de escrever: `change_sequence` é a versão da linha na chave (1 INSERT, 2 UPDATE, 3 DELETE); dentro de cada par (chave, versão) o conteúdo é idêntico, então não há empate; a linha `DELETE` chega com todos os atributos nulos, e por isso o cadastro completo só é exigido quando a operação não é `DELETE`. Validação ganhou as verificações 11 (`cdc_ultima_versao`) e 12 (`cdc_exclusoes_marcadas`). Notebook `03_documentacao_pii` aplica comentários de coluna e tags `pii`, `pii_categoria` e `contem_pii`.
- **2026-11-12 — etapa 2.5 concluída:** execução normal do pipeline: `silver.customers` 98.476 linhas (288 com `operation = 'DELETE'`) e `silver.sellers` 3.095 (21 `DELETE`), uma linha por chave, quarentenas vazias; as outras 8 entidades inalteradas. Balanço de `customers` (106.667 = 98.476 + 0 + 8.191 duplicatas) e de `sellers` (3.430 = 3.095 + 0 + 335). `dq_resultados` passou a ter as 10 entidades. Comentários em 13 de 13 colunas (`customers`) e 12 de 12 (`sellers`); 8 e 6 tags de coluna e 1 tag de tabela em cada. Depois de uma execução normal e de um *full refresh*, tags e comentários continuaram nas tabelas. Validação `c41a58bd`: **55 OK, 0 reprovadas**, idempotência `ok = true` nas 10 entidades.

## Decisões

- **`sequence_by` de `order_payments` = `sim_date`** (2.2): a entidade não tem timestamp de negócio e as duplicatas são cópias idênticas, então a exceção à regra "nunca `sim_date`" (S3) não altera o resultado. Registrar no ADR da fase.
- **Descrição das entidades por dados + `registrar_entidade`** (2.2): evita repetir o mesmo fluxo 8 vezes; nova entidade = nova linha. Na 2.5, `customers` e `sellers` entraram só com duas linhas.
- **Corrigir e marcar em vez de quarentenar** `price` com vírgula decimal (D1, 2.3).
- **`DELETE` fica na silver** (2.5): a silver SCD1 guarda a última versão com `operation`; histórico e tratamento da exclusão ficam para as Fases 3 e 6.
- **Funções puras ficam no próprio notebook** (desvio do desenho S1): o caminho de importação de módulo no pipeline não foi verificado; extração para módulo na Fase 7.

Todas as decisões da fase estão consolidadas em [ADR 0005](../adr/0005-silver-declarativa-deduplicacao-e-quarentena.md).

## Problemas encontrados e como foram resolvidos

| Problema | Causa | Solução |
|---|---|---|
| `_dq_correcao` nula nos 4 itens corrigidos (2.3) | `corrigir_formato` calculava a marca num segundo `withColumn`, depois da troca do `price`; naquele ponto a coluna já não era nula e a condição "estava nula" dava falso | Marca calculada no mesmo `withColumns` da troca, onde todas as expressões enxergam as colunas originais. Exige novo full refresh de `order_items` |
| Primeira execução falhou em 10 s com `NO_TABLES_IN_PIPELINE` (2.1) | O pipeline foi criado com o código-fonte padrão (`transformations/my_transformation.py`, vazio); o notebook importado não estava na lista de fontes | Trocar o código-fonte do pipeline pelo notebook `src/silver/01_silver_pipeline`. Nenhuma tabela nem checkpoint foi criado |

## Aprendizados

- Pipeline novo nasce com uma pasta `transformations/` de exemplo: é preciso apontar o código-fonte para o notebook do repositório.
- Na interface do editor de pipelines, o "validate" chama-se **Dry run**: valida o grafo sem processar dados.
- Tabela de quarentena vazia não indica defeito; indica que nenhuma falha daquela entidade chegou à regra (conferir contra o gabarito).
- O pipeline mede a si mesmo (`dq_resultados`); só a validação independente contra a landing e o gabarito prova que a silver está certa. Testes de contagem não pegam uma marca nula (`_dq_correcao`): é preciso conferir o conteúdo.
- Olhar os dados antes de escrever a regra evita erro de desenho: foi assim que apareceu o `DELETE` com atributos nulos, que teria mandado 304 clientes e 22 vendedores para a quarentena.
- `ALTER TABLE` (comentário e tag) funciona em streaming tables do pipeline e sobrevive a execuções e a *full refresh*. Uma execução normal sem dados novos é um teste fraco; o teste que vale é o refresh.
- Uma tabela nova no pipeline entra com execução normal; *full refresh* só quando uma regra nova precisa valer para o histórico.

## Encerramento — pendências para a Fase 3

**Resultado dos critérios de aceite**
- Silver tipada e deduplicada, validada contra o gabarito (`corrupt_file`, `bad_value`, `new_column`, `late_arrival`, `out_of_order`, duplicatas): ✅ 55 verificações OK, 0 reprovadas.
- Rodar o pipeline duas vezes sem mudar o resultado: ✅ idempotência `ok` nas 10 entidades.
- Quarentena com motivo e valor fora do padrão corrigido e marcado: ✅.
- Tabelas documentadas, tags de PII nos cadastros e ADR da decisão principal: ✅ (ADR 0005).

**Estado entregue**
- `olist.silver` com 10 tabelas: `orders` 98.476, `order_items` 111.592, `order_payments` 102.906, `order_reviews` 95.253, `order_status_events` 387.691, `products` 32.951, `geolocation` 1.000.163, `category_translation` 71, `customers` 98.476, `sellers` 3.095.
- `olist.quality`: 10 tabelas de quarentena (só `order_items` com 1 linha), `dq_resultados` e `validacao_resultados`.
- Notebooks em `src/silver/` (pipeline) e `src/quality/` (`01_dq_resultados`, `02_validacao_silver`, `03_documentacao_pii`).

**Para o CDC e o SCD2 (Fase 3)**
- Construir o histórico de `customers` e `sellers` a partir do feed (`operation`, `change_sequence`, `change_ts`); a silver SCD1 guarda só a última versão.
- Decidir o que fazer com as exclusões: 288 clientes e 21 vendedores têm `DELETE` como última versão, e a linha `DELETE` chega sem atributos (fica na silver com `operation = 'DELETE'`).
- `change_ts` e `sim_date` do cadastro chegam a 2028, porque a carga histórica avançou além do presente: o SCD2 deve ordenar pelo tempo do negócio e tratar a validade das versões com atenção a isso.
- Decidir como a gold e as dimensões tratam pedidos de clientes excluídos (a Fase 6 trata a exclusão auditável).

**Para a Fase 4 (gold)**
- 1.370 eventos de `order_status_events` têm `event_ts` anterior ao evento de sequência menor da mesma ordem (provavelmente do dataset original): a métrica "entregue no prazo" não pode supor timestamps crescentes por `event_sequence`.
- `payment_gateway` é nula antes de 2026-10-08 (137 pagamentos medidos como aviso): coluna opcional.

**Pendências técnicas**
- [ ] Fase 7: testes unitários (pytest) de `corrigir_formato` e `montar_regras`; a marca nula em `_dq_correcao` passou despercebida pelos testes de contagem.
- [ ] Fase 7: mover as funções puras e a chave de cada entidade (hoje repetidas nos notebooks de pipeline e de validação) para um módulo único, depois de verificar como o pipeline resolve o caminho de importação.
- [ ] Fase 7: orquestrar pipeline → `01_dq_resultados` → `02_validacao_silver` → `03_documentacao_pii` num job; a validação já falha no fim se algo reprovar.
- [ ] Fase 6: política de PII por categoria (`pii_categoria`) e exclusão auditável.
- [ ] Configurar a proteção da `main` no GitHub (herdada das Fases 0 e 1).
- [ ] Os últimos dois meses do dataset original seguem fora da simulação (Fase 5).
