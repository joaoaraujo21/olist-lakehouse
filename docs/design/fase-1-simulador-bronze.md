# Documento de Desenho Técnico — Fase 1: Simulador e Bronze

| | |
|---|---|
| **Projeto** | Olist Lakehouse |
| **Fase** | 1 — Simulador e Bronze |
| **Responsável técnico** | João Guilherme Araujo |
| **Data** | 2026-11-02 |
| **Status** | Em revisão |
| **Revisões** | 2026-11-07 — abertura da etapa 1.4: A5 detalhado; incluídos A6 (feed de mudanças cadastrais) e A7 (reinício controlado da simulação); 2026-11-08 — A5: função de hash do sorteio, gravação do gabarito e resultado observado da linha de JSON inválido; abertura da etapa 1.5: coluna `_corrupt_record` na bronze (ADR 0004) |
| **Documentos relacionados** | [ADR 0001 — Organização da landing](../adr/0001-organizacao-landing.md) · [ADR 0002 — Pedido válido e entrega no prazo](../adr/0002-pedido-valido-e-entrega-no-prazo.md) · [Dicionário de métricas v0](../dicionario_metricas.md) |

---

## 1. Objetivo deste documento

Registrar as decisões técnicas da Fase 1, tomadas após o alinhamento com a área de negócio, **antes** do início da implementação. O documento responde às perguntas de desenho levantadas na abertura da fase e serve de referência para a implementação, para o code review e para os ADRs que serão publicados ao final da fase.

## 2. Contexto e alinhamento com o negócio

A operação da Olist gera pedidos continuamente. Os sistemas de origem (checkout, pagamentos, logística e cadastro) exportam arquivos para a nossa zona de landing em intervalos regulares. Esses sistemas apresentam falhas conhecidas, que a plataforma de dados precisa suportar sem perder nem duplicar informação:

| Falha da origem | Exemplo |
|---|---|
| Reenvio | o mesmo arquivo é enviado duas vezes após um timeout |
| Atraso | um lote de ontem chega somente hoje |
| Fora de ordem | o evento "entregue" chega antes do evento "despachado" |
| Mudança de schema | o time de pagamentos adiciona uma coluna sem aviso prévio |
| Mudança cadastral | vendedores e clientes alteram endereço, e o cadastro envia as alterações |

Como o dataset da Olist é histórico (2016–2018), ficou alinhado que a fase entrega **dois componentes**:

1. **Simulador de operação:** faz o papel dos sistemas de origem. Lê o dataset original e o "reencena" no presente, gravando arquivos diários na landing, inclusive com as falhas acima.
2. **Camada bronze:** captura de forma incremental tudo o que chega na landing e guarda o dado bruto, exatamente como foi recebido, com rastreabilidade completa.

**Requisitos acordados com o negócio:**
- Nenhum dado recebido pode ser perdido.
- Reexecutar qualquer etapa não pode duplicar dados.
- Deve ser possível responder, para qualquer registro: de qual arquivo veio, quando chegou e qual execução o gravou.
- A operação precisa caber na cota do ambiente (Databricks Free Edition).

## 3. Escopo

**Dentro do escopo:** simulador (carga inicial, eventos diários, falhas injetadas, feed de mudanças cadastrais), tabela de controle do simulador, ingestão bronze de todas as entidades, colunas de auditoria e testes de aceite de idempotência e de evolução de schema.

**Fora do escopo (fases seguintes):** limpeza, tipagem e deduplicação (Fase 2), aplicação do CDC/SCD2 (Fase 3), orquestração por job agendado (Fase 7).

## 4. Visão geral da solução

```
olist.landing.olist_source  (9 CSVs originais, somente leitura)
        │
        │  SIMULADOR — cada execução processa N dias simulados
        ▼
olist.landing.raw_data/<entidade>/sim_date=AAAA-MM-DD/<entidade>_AAAA-MM-DD_part-000.json
        │
        │  AUTO LOADER — 1 stream por entidade, trigger availableNow
        ▼
olist.bronze.<entidade>  (Delta, append-only, colunas como string + auditoria)

Controle e estado:
  olist.ops.simulator_runs         progresso do simulador (watermark)
  olist.ops.simulator_chaos_log    registro das falhas injetadas (gabarito)
  /Volumes/olist/ops/checkpoints/  checkpoints e schemas dos streams
```

---

## 5. Perguntas e respostas de desenho

### Bloco A — Simulador

#### A1. Relógio da simulação

**Pergunta:** Como as datas históricas (2016–2018) serão convertidas em datas atuais?

**Resposta:** Todas as colunas de data e hora do dataset serão deslocadas por um valor fixo de **3.654 dias (522 semanas)**, definido em uma constante de configuração (`DATE_OFFSET_DAYS`). O mesmo deslocamento é aplicado a todas as colunas de data: compra, aprovação, despacho, entrega, data estimada, data limite de envio e datas da avaliação.

| Data original | Data simulada |
|---|---|
| 2016-09-04 (domingo) — primeiro pedido | 2026-09-06 (domingo) |
| 2017-01-01 — início do volume regular de pedidos | 2027-01-03 |
| 2017-11-24 (sexta) — Black Friday | 2027-11-26 (sexta) — Black Friday |
| 2018-10-17 — último pedido | 2028-10-18 |

Justificativa:
- O deslocamento é **múltiplo de 7 dias** e por isso preserva o dia da semana de cada evento. Padrões sazonais (menos vendas aos domingos, pico na Black Friday) se mantêm nas análises.
- Por ser o mesmo para todas as colunas, o deslocamento mantém intactas as **durações** (lead time, dias de atraso, prazo de entrega). As métricas calculadas sobre os dados simulados são idênticas às do dataset original, o que permite conciliar os números.

Alternativas avaliadas:
- *Somar exatamente 10 anos:* descartada, porque altera o dia da semana e gera inconsistência em 29 de fevereiro.
- *Ancorar o primeiro pedido na data de hoje:* descartada, porque o resultado mudaria conforme o dia da execução, o que impede a reprodutibilidade.

Ponto de atenção: as datas simuladas ficam à frente da data real. Por isso, **a referência de "hoje" do negócio é a data simulada corrente**, registrada na tabela de controle (A1.3), e nunca a data do sistema (`current_date()`). Essa regra vale para todas as fases, em especial para o monitor de atrasos da Fase 5.

---

**Pergunta:** Quanto tempo simulado cada execução avança?

**Resposta:** A unidade do relógio é o **dia**. Cada dia simulado gera seu próprio conjunto de arquivos. Uma execução processa uma quantidade configurável de dias consecutivos, definida pelo parâmetro `dias_por_execucao`:

| Modo de operação | `dias_por_execucao` | Finalidade |
|---|---|---|
| Desenvolvimento e testes | 1 | validar a geração de um dia por vez |
| Carga histórica | 30 | acumular rapidamente o histórico necessário às fases 2 a 4 |
| Operação | 1 | execução agendada: um dia de negócio por execução |

Justificativa: o dataset cobre aproximadamente 773 dias. Avançar somente um dia por execução exigiria 773 execuções para completar o histórico, o que é inviável dentro da cota do ambiente. Como a granularidade dos arquivos continua diária, o resultado final é o mesmo em qualquer modo: muda apenas quantos dias cada execução processa.

---

**Pergunta:** Onde fica registrado até que ponto o simulador já avançou?

**Resposta:** Em uma tabela Delta de controle, `olist.ops.simulator_runs`, com uma linha por execução do simulador:

| Coluna | Descrição |
|---|---|
| `run_id` | identificador único da execução, também gravado nas tabelas bronze |
| `sim_date_start`, `sim_date_end` | intervalo de dias simulados processado |
| `status` | `RUNNING`, `SUCCESS` ou `FAILED` |
| `started_at`, `finished_at` | horários reais de início e fim |
| `files_written`, `rows_written` | volume gerado na execução |

Funcionamento:
1. A execução consulta o último dia processado com sucesso (`MAX(sim_date_end) WHERE status = 'SUCCESS'`). Se não houver registro, inicia em `2026-09-06`.
2. Registra a execução como `RUNNING` e gera os dias seguintes, conforme `dias_por_execucao`.
3. Ao final, atualiza o status para `SUCCESS` ou, em caso de erro, para `FAILED`.

Uma execução com falha não altera o progresso; a execução seguinte reprocessa os mesmos dias. Como os arquivos têm nome e conteúdo determinísticos (A3), esse reprocessamento não gera duplicidade. Para reprocessar um dia específico (backfill), a execução aceita o parâmetro opcional `sim_date_override`.

Justificativa: esse é o padrão de **tabela de controle de carga (watermark)**, amplamente adotado em ingestão incremental. O schema `ops` foi criado na Fase 0 justamente para concentrar o estado operacional da plataforma.

Alternativas avaliadas:
- *Inferir o progresso listando os arquivos da landing:* descartada. Um arquivo parcial ou um reenvio distorceria o cálculo, e dado de negócio se misturaria com controle.
- *Arquivo de estado em um Volume:* descartada. Não tem controle transacional nem histórico; a tabela Delta oferece os dois e pode ser consultada via SQL.

#### A2. Entidades e forma de emissão

**Pergunta:** Quais tabelas do dataset são transacionais (chegam ao longo do tempo) e quais são cadastrais ou estáticas?

**Resposta:** As 9 tabelas originais dão origem a 10 entidades na landing:

| Entidade na landing | Tabela de origem | Classificação | Momento de emissão (dia simulado) |
|---|---|---|---|
| `orders` | orders | transacional | dia da compra (somente o cabeçalho do pedido) |
| `order_status_events` | orders (derivada) | transacional — eventos | dia de cada mudança de status |
| `order_items` | order_items | transacional | dia da compra |
| `order_payments` | order_payments | transacional | dia da compra |
| `order_reviews` | order_reviews | transacional | dia da resposta da avaliação |
| `customers` | customers | cadastral com feed de mudanças (CDC) | inclusão no dia do primeiro pedido do cliente; alterações sintéticas ao longo do tempo |
| `sellers` | sellers | cadastral com feed de mudanças (CDC) | carga inicial completa no primeiro dia; alterações e exclusões sintéticas ao longo do tempo |
| `products` | products | cadastral | carga inicial no primeiro dia |
| `geolocation` | geolocation | referência estática | carga única no primeiro dia |
| `category_translation` | product_category_name_translation | referência estática | carga única no primeiro dia |

Os feeds de `customers` e `sellers` seguem o formato de CDC (Change Data Capture). Além dos atributos, cada registro traz `operation` (`INSERT`, `UPDATE` ou `DELETE`), `change_ts` (momento da alteração) e `change_sequence` (ordem da alteração). Esse feed será consumido pela Fase 3 (histórico SCD2), e as exclusões de clientes vão alimentar o processo de direito ao esquecimento da Fase 6 (LGPD).

O cliente é emitido junto com o seu primeiro pedido porque, no modelo da Olist, o registro de cliente nasce no checkout: cada pedido tem um `customer_id` próprio, ligado ao identificador permanente `customer_unique_id`.

---

**Pergunta:** Como será representada a evolução de status dos pedidos, considerando que o dataset traz um único registro por pedido?

**Resposta:** O registro original de pedido será dividido em duas entidades:

- **`orders` (cabeçalho do pedido):** contém apenas as informações conhecidas no momento da compra: `order_id`, `customer_id`, `order_purchase_timestamp` e `order_estimated_delivery_date`. A data estimada é informada ao cliente no checkout.
- **`order_status_events` (eventos de status):** um registro por mudança de status, com `order_id`, `status`, `event_ts` e `event_sequence`. Cada evento é emitido no dia simulado em que ocorreu:

| Evento | Coluna de origem | Exemplo de emissão |
|---|---|---|
| `created` | `order_purchase_timestamp` | dia D |
| `approved` | `order_approved_at` | dia D |
| `shipped` | `order_delivered_carrier_date` | dia D+2 |
| `delivered` | `order_delivered_customer_date` | dia D+9 |

Regras complementares:
- Pedidos com status final `canceled`, `unavailable`, `invoiced` ou `processing` recebem um evento final com esse status, datado na última data conhecida do pedido.
- Eventos com data posterior ao fim do dataset não são emitidos. Esses pedidos permanecem em aberto, como ocorreria na operação real.

Justificativa: o registro original é uma fotografia do estado final do pedido. Na operação real, ninguém conhece no dia da compra a data em que o pedido será entregue: essa informação chega aos poucos. Modelar os pedidos como eventos obriga a plataforma a reconstruir o estado atual de cada pedido a partir do histórico, que é o desafio central da Fase 5. Também torna possível identificar pedidos despachados e ainda não entregues, que são a base do monitor de atrasos.

Alternativa avaliada: *reemitir o registro completo do pedido a cada mudança (snapshots sucessivos).* Descartada por gerar redundância e não representar o modelo de eventos que a Fase 5 exige.

#### A3. Formato e nomenclatura dos arquivos

**Pergunta:** Qual será o formato dos arquivos: CSV ou JSON?

**Resposta:** **JSON Lines**: um objeto JSON por linha, extensão `.json`.

| Critério | CSV | JSON Lines |
|---|---|---|
| Texto com vírgulas, aspas ou quebras de linha (comentários das avaliações) | exige opções específicas de leitura (`multiLine`, `escape`), como observado na Fase 0 | suportado nativamente |
| Inclusão ou reordenação de colunas | depende da posição e do cabeçalho | cada campo é identificado pelo nome |
| Detecção de mudança de schema | limitada | o Auto Loader identifica campos novos e evolui o schema |
| Aderência a eventos e CDC | baixa | formato padrão de eventos e mensageria |
| Tamanho | menor | maior, pois repete o nome dos campos; irrelevante no volume do projeto |

Justificativa: o JSON elimina uma classe inteira de problemas de leitura e é o formato natural para eventos e feeds de mudança. O custo adicional de armazenamento não é relevante para cerca de 100 mil pedidos.

---

**Pergunta:** Qual será o padrão de nome e de caminho dos arquivos?

**Resposta:**

```
/Volumes/olist/landing/raw_data/<entidade>/sim_date=<AAAA-MM-DD>/<entidade>_<AAAA-MM-DD>_part-000.json

Exemplo:
/Volumes/olist/landing/raw_data/orders/sim_date=2026-09-06/orders_2026-09-06_part-000.json
```

- **Uma pasta por entidade**, conforme o ADR 0001: cada stream de ingestão lê uma única pasta, com um único schema.
- **Subpasta `sim_date=AAAA-MM-DD`** no padrão de particionamento Hive: o Auto Loader extrai o dia simulado como coluna automaticamente, e cada dia pode ser localizado, auditado ou reprocessado isoladamente.
- **O nome depende somente da entidade, do dia simulado e do número da parte.** Não contém data e hora de execução nem identificadores aleatórios, o que garante a idempotência descrita a seguir.
- O sufixo `part-000` permite dividir um dia em vários arquivos no futuro sem alterar o padrão.

**Escrita atômica:** o simulador grava cada arquivo primeiro em uma pasta de preparação (`raw_data/_staging/`) e, ao concluir, move o arquivo para o destino final. Assim o Auto Loader nunca lê um arquivo incompleto. Por padrão, o Auto Loader ignora pastas e arquivos cujo nome começa com `_` ou `.`.

---

**Pergunta:** O que garante que executar o simulador várias vezes sobre o mesmo dia não gere duplicidade na bronze?

**Resposta:** A combinação de três garantias:

1. **Caminho determinístico:** o mesmo dia simulado gera sempre o mesmo caminho de arquivo.
2. **Conteúdo determinístico:** os registros são ordenados por chave, e as falhas injetadas usam semente fixa (A5). Gerar o mesmo dia em momentos diferentes produz arquivos idênticos.
3. **Controle de arquivos do Auto Loader:** o Auto Loader registra no checkpoint cada arquivo já processado. Com a opção `cloudFiles.allowOverwrites = false` (valor padrão), um arquivo regravado no mesmo caminho não é processado novamente.

Na prática, quando um dia é regerado (falha, backfill ou teste), o arquivo é sobrescrito com o mesmo conteúdo e ignorado pela ingestão. O resultado é **zero duplicidade técnica**.

Restrição de implementação: é proibido usar `uuid`, data e hora de execução ou qualquer valor variável no nome dos arquivos. Cada execução geraria nomes novos, o Auto Loader os trataria como arquivos inéditos e os mesmos dados seriam ingeridos novamente.

#### A4. Tamanho dos lotes

**Pergunta:** Quantos dias simulados cada execução deve processar?

**Resposta:** Um dia por execução nos modos de desenvolvimento e de operação, e 30 dias por execução na carga histórica, conforme definido em A1.

Justificativa: no ambiente Free Edition, o custo está no **número de execuções** (cada execução aloca compute) e não no volume de dados. O dataset completo tem cerca de 126 MB e é processado em segundos. A estratégia, portanto, é fazer poucas execuções grandes na carga histórica e uma execução diária na operação.

---

**Pergunta:** Quantos arquivos cada entidade gera por dia simulado?

**Resposta:** **Um arquivo por entidade por dia simulado.** A média é de cerca de 130 pedidos por dia, com pico de cerca de 1.100 na Black Friday. Cada arquivo diário tem entre poucos KB e cerca de 1 MB, sem necessidade de divisão. Ao longo de toda a simulação, estima-se cerca de 5.400 arquivos.

Justificativa:
- Esse volume de arquivos de entrada é adequado ao Auto Loader.
- Do lado das tabelas, o catálogo `olist` está com **Predictive Optimization** ativa (herdada do metastore), que executa compactação de arquivos automaticamente. O impacto de arquivos pequenos será medido na Fase 8.
- Granularidade horária foi descartada: multiplicaria o número de arquivos por 24 sem benefício para as análises previstas.

#### A5. Injeção de falhas

**Pergunta:** Como serão simuladas as falhas da origem (duplicidade, atraso, fora de ordem e mudança de schema)?

**Resposta:**

| Falha | Implementação no simulador | Taxa inicial | Camada responsável pelo tratamento |
|---|---|---|---|
| Reenvio de arquivo | cópia do arquivo do dia gravada com outro nome (`..._part-000_retry-1.json`) | 5% dos arquivos | silver — deduplicação |
| Registro duplicado | repetição de registros dentro do mesmo arquivo | 1% dos registros | silver — deduplicação |
| Chegada atrasada | registro do dia D gravado no arquivo de D+1 a D+3, mantendo o `event_ts` original | 3% dos registros | silver — ordenação por data do evento |
| Fora de ordem | decorrência do atraso nos eventos de status (o `delivered` chega antes do `shipped`), mais o embaralhamento das linhas nos arquivos de eventos | — | silver — ordenação por `event_sequence` |
| Coluna nova | a partir de uma data configurada, `order_payments` passa a conter `payment_gateway` | uma data fixa | bronze — evolução de schema |
| Valor fora do padrão | alguns valores de `price` emitidos com vírgula decimal (`"12,90"`) | uma data fixa, poucos registros | silver — tipagem e quarentena |
| Arquivo corrompido | uma linha de JSON inválido em um arquivo | uma data fixa, um arquivo | bronze — texto original da linha em `_corrupt_record` (ADR 0004) |

Os reenvios usam, de propósito, um **nome diferente** do arquivo original: representam a origem enviando o mesmo dado duas vezes, que é uma **duplicidade de negócio** e deve ser preservada na bronze (B6). A duplicidade técnica, causada pela reexecução do nosso próprio processo, é impedida pelas garantias de A3.

Todas as taxas e datas ficam centralizadas em uma configuração única (`CHAOS_CONFIG`), com um indicador `enabled`. A etapa 1.2 gera dados sem falhas; a etapa 1.4 ativa as falhas sem alteração de código.

---

**Pergunta:** Como garantir que as falhas injetadas sejam reproduzíveis?

**Resposta:** Por meio de **aleatoriedade determinística**:
- Decisões por arquivo ou por dia usam um gerador com semente derivada da data simulada: `random.Random(f"{SEED_GLOBAL}-{sim_date}")`.
- Decisões por registro usam o hash da chave do registro combinado com a semente e com o tipo de falha (por exemplo, `pmod(xxhash64(semente, "late_arrival", order_id, order_item_id), 100) < 3`). Assim, o mesmo registro recebe sempre o mesmo tratamento, independentemente da ordem de processamento, e as falhas são independentes entre si. A função `hash()` do Python não é utilizada: para textos, seu resultado muda a cada processo.

Cada falha injetada é registrada na tabela `olist.ops.simulator_chaos_log` (`run_id`, `sim_date`, `chaos_type`, `entity`, `record_key`, `detail`, `logged_at`), que funciona como **gabarito**. A gravação substitui os dias reprocessados em uma única transação (`replaceWhere` por intervalo de `sim_date`). Na Fase 2, os testes da silver comparam o resultado da deduplicação com esse gabarito, comprovando que exatamente as duplicatas injetadas foram removidas.

Justificativa: falhas puramente aleatórias tornariam cada geração de um mesmo dia diferente da anterior, o que quebra a idempotência (A3) e impede a reprodução de defeitos encontrados nas camadas seguintes.

---

**Pergunta:** Em quais entidades e em quais datas cada falha será aplicada?

**Resposta:** Definido na abertura da etapa 1.4, a partir da distribuição dos pedidos simulados. Os dias de 2026-09-06 a 2026-10-04 têm somente 5 pedidos; o primeiro volume real ocorre de **2026-10-05 a 2026-10-12** (322 pedidos), e a primeira avaliação respondida é de 2026-10-09. As falhas de data fixa ficam nesse intervalo, em dias diferentes, para que o efeito de cada uma possa ser observado isoladamente:

| Falha | Entidades | Data ou regra |
|---|---|---|
| Reenvio de arquivo | todas as entidades diárias | 5% dos arquivos (sorteio por entidade e dia) |
| Registro duplicado | entidades transacionais e feeds CDC; exceto cargas estáticas | 1% dos registros |
| Chegada atrasada | `order_status_events` e `order_items` | 3% dos registros, atraso de 1 a 3 dias; registros que ultrapassariam o último dia da simulação não são atrasados |
| Fora de ordem | `order_status_events` | decorrência do atraso; linhas dos arquivos de eventos ordenadas por hash da chave (ordem embaralhada, porém idêntica a cada geração) |
| Coluna nova `payment_gateway` | `order_payments` | a partir de **2026-10-08** |
| `price` com vírgula decimal | `order_items` | **2026-10-07**, poucos registros |
| Linha de JSON inválido | `order_items` | **2026-10-10**, um arquivo |

Regras complementares:
- **O cabeçalho do pedido (`orders`) nunca atrasa.** Itens e eventos podem chegar depois do pedido, mas nunca antes. A silver da Fase 2 trata atraso e fora de ordem sem precisar tratar registros órfãos, tema reservado à Fase 5.
- **Os feeds CDC não atrasam.** A ordem das mudanças é garantida por `change_sequence` (A6); atrasá-los misturaria dois testes na Fase 3.
- **A duplicação não se aplica às cargas estáticas.** `geolocation` já contém linhas repetidas na origem, e misturar duplicatas naturais e injetadas inviabilizaria o gabarito.
- A linha inválida não fica no arquivo de `order_reviews` de 2026-10-09, que é o primeiro dessa entidade e é usado na inferência de schema.
- O gabarito de um dia é substituído quando o dia é reprocessado, como acontece com os arquivos da landing, de modo que o gabarito sempre corresponde ao conteúdo atual da landing.

#### A6. Feed de mudanças cadastrais (CDC)

**Pergunta:** Quais mudanças cadastrais o simulador emite, com que frequência e em que formato?

**Resposta:** O dataset não contém histórico cadastral; as mudanças são **sintéticas** e determinísticas. Cada cliente ou vendedor tem, desde o início, a definição de se e quando vai mudar, calculada por hash da sua chave, como nas falhas de A5:

| Feed | Operação | Abrangência | Momento (dia simulado) |
|---|---|---|---|
| `customers` | `UPDATE` — mudança de endereço | 3% dos clientes | inclusão + 7 a 120 dias |
| `customers` | `DELETE` — solicitação de exclusão (LGPD) | 0,5% dos clientes | última data conhecida do pedido + 30 a 365 dias |
| `sellers` | `UPDATE` — mudança de endereço | 10% dos vendedores | qualquer dia após a carga inicial |
| `sellers` | `DELETE` — desligamento | 2% dos vendedores | última venda + 30 a 90 dias |

Regras:
- **Atributos alterados:** somente o endereço (CEP, cidade e UF). O endereço novo é escolhido, por hash, entre endereços existentes na base da própria entidade, o que evita combinações inválidas de CEP, cidade e UF.
- **Conteúdo do registro:** o `UPDATE` traz o estado completo após a alteração. O `DELETE` traz apenas as chaves, com os demais atributos nulos: um evento de exclusão não deve reenviar os dados pessoais cuja exclusão foi solicitada.
- **Ordem:** `change_sequence` 1 (`INSERT`), 2 (`UPDATE`) e 3 (`DELETE`). Uma mudança nunca ocorre após a exclusão: se a data sorteada para o `UPDATE` não for anterior à do `DELETE`, o `UPDATE` não é emitido. Mudanças com data posterior ao último dia da simulação não são emitidas.
- **Desligamento após a última venda:** nenhum pedido referencia um vendedor já excluído, o que seria uma inconsistência produzida pela simulação e não um comportamento da origem.
- As mudanças cadastrais são dados legítimos e **não** são registradas em `ops.simulator_chaos_log`, que contém somente as falhas injetadas.

Justificativa: endereço é o atributo que efetivamente muda no cadastro de clientes e vendedores e é o que afeta as análises por região, cenário que motiva o histórico SCD2 da Fase 3. As taxas representam uma operação estável e garantem mudanças de vendedores já em outubro de 2026, dentro do período gerado na Fase 1; as exclusões de clientes (cerca de 500 ao longo da simulação) são suficientes para o processo de direito ao esquecimento da Fase 6.

#### A7. Reinício controlado da simulação

**Pergunta:** Como tratar os dias gerados antes da ativação das falhas?

**Resposta:** Os dias 2026-09-06 e 2026-09-07 foram gerados e ingeridos sem falhas. Regerá-los com as falhas ativas produziria arquivos diferentes com o mesmo nome, que o Auto Loader não processaria novamente (`allowOverwrites = false`), deixando a landing e a bronze divergentes. Por isso, a simulação será **reiniciada** desde o primeiro dia, com as falhas ativas.

O reinício é feito pelo notebook `setup/99_reset_simulacao`, que apaga, nesta ordem: tabelas bronze das entidades, checkpoints da bronze, arquivos da landing (`raw_data`) e registros de `ops.simulator_runs` e `ops.simulator_chaos_log`. A estrutura (schemas, volumes e tabelas de controle) é mantida. O notebook exibe o que será apagado e só executa a exclusão com a confirmação explícita no parâmetro `confirmacao`, que inclui a data do dia (`APAGAR AAAA-MM-DD`): como o valor do parâmetro fica salvo no notebook, uma confirmação esquecida deixa de valer no dia seguinte.

Justificativa:
- Aplica a regra operacional de B3: o reprocessamento completo é uma operação intencional, que apaga checkpoint e tabela na mesma operação. Apagar somente parte do estado causaria perda ou duplicidade de dados.
- Um procedimento destrutivo deve ser versionado, revisável e repetível, e não depender de comandos manuais.
- O custo é mínimo enquanto a simulação tem poucos dias gerados.

Alternativa avaliada: *ativar as falhas somente a partir de uma data, preservando os dias já gerados.* Descartada porque a simulação passaria a ter um período inicial sem falhas que não representa a origem, distorcendo as taxas medidas na silver.

### Bloco B — Bronze

#### B1. Tecnologia de ingestão

**Pergunta:** A ingestão bronze será implementada em notebook com Structured Streaming ou em pipeline Lakeflow Declarative?

**Resposta:** **Notebook com Structured Streaming e Auto Loader**, organizado em uma função genérica de ingestão (`ingest_entity`) que recebe o nome da entidade e deriva dela o caminho de origem, o checkpoint e a tabela de destino. Incluir uma nova entidade consiste em adicionar uma linha à configuração.

Justificativa:
- **Domínio técnico:** a implementação em notebook exige configurar explicitamente checkpoint, local de schema, trigger, evolução de schema e metadados de arquivo, que são fundamentos do Auto Loader e conteúdo central da certificação Data Engineer Associate.
- **Limite do ambiente:** a Free Edition permite um pipeline Lakeflow ativo por tipo. A silver (Fase 2) será implementada em Lakeflow Declarative Pipelines; manter a bronze em notebook evita disputar esse recurso.
- **Desacoplamento:** ingestão e transformação ficam independentes. A silver pode ser reprocessada sem afetar a bronze, e a bronze continua capturando dados mesmo que a silver apresente falhas.

Alternativa avaliada: *bronze em pipeline Lakeflow (streaming tables com Auto Loader).* É a abordagem recomendada pela Databricks para pipelines novos, pois gerencia checkpoints, reprocessamento e linhagem. Não foi adotada nesta fase pelos motivos acima. A comparação entre as duas abordagens será registrada na Fase 8.

#### B2. Modo de execução

**Pergunta:** A ingestão será contínua ou processará os arquivos disponíveis e encerrará?

**Resposta:** **`trigger(availableNow=True)`.** O stream processa todos os arquivos recebidos desde a última execução, em quantos micro-batches forem necessários, e encerra automaticamente.

Justificativa:
- **Compatibilidade:** pela documentação de limitações do compute serverless, notebooks e jobs serverless suportam `AvailableNow` (recomendado) e `Once` (obsoleto). Os modos `ProcessingTime` e `Continuous` não são suportados e geram o erro `INFINITE_STREAMING_TRIGGER_NOT_SUPPORTED`. Sem trigger declarado, o Spark assume `ProcessingTime("0 seconds")`, o que também gera erro. **O trigger deve ser sempre declarado explicitamente.**
- **Custo:** o compute é alocado somente durante o processamento. Um stream contínuo consumiria a cota do ambiente de forma permanente.
- **Flexibilidade:** o código mantém a semântica de streaming (checkpoint e processamento incremental). A frequência de execução passa a ser apenas uma questão de agendamento, definida na Fase 7 com o encadeamento `simulador → bronze`.

#### B3. Checkpoint e local de schema

**Pergunta:** Onde ficarão o checkpoint e o local de armazenamento do schema inferido de cada stream?

**Resposta:** Em um Volume dedicado, **`olist.ops.checkpoints`**, com uma pasta exclusiva por stream:

```
/Volumes/olist/ops/checkpoints/bronze/<entidade>/            checkpointLocation
/Volumes/olist/ops/checkpoints/bronze/<entidade>/_schema/    cloudFiles.schemaLocation
```

Justificativa:
- No compute serverless, o armazenamento persistente de arquivos é feito em Volumes do Unity Catalog.
- Checkpoint é estado operacional e, portanto, pertence ao schema `ops`, separado dos dados de negócio.
- Cada stream tem um checkpoint exclusivo. O compartilhamento de checkpoint entre streams corrompe o estado de ambos.
- Manter o local de schema dentro da pasta do checkpoint segue o padrão da documentação oficial e concentra todo o estado de um stream em um único lugar.

Regra operacional: o checkpoint registra os arquivos já processados e é o que garante que cada arquivo seja ingerido uma única vez. **Apagar o checkpoint faz o Auto Loader reprocessar todos os arquivos**, o que duplica os dados na bronze, já que ela é append-only. O reprocessamento completo de uma entidade só pode ser feito de forma intencional, apagando o checkpoint e recriando a tabela bronze correspondente na mesma operação.

Pré-requisito: incluir a criação do Volume `olist.ops.checkpoints` (com `IF NOT EXISTS` e `COMMENT`) no notebook de setup.

#### B4. Tratamento de schema

**Pergunta:** O schema da bronze será inferido ou declarado explicitamente?

**Resposta:** **Inferido pelo Auto Loader, com todas as colunas armazenadas como `string`.** Esse é o comportamento padrão do Auto Loader para JSON (`cloudFiles.inferColumnTypes = false`). A conversão para os tipos corretos ocorre na silver.

Justificativa:
- A bronze preserva o dado exatamente como foi recebido. Um valor como `"12,90"` permanece registrado, mantendo a evidência do problema da origem.
- A ingestão não falha por incompatibilidade de tipo. Problemas de tipo são tratados e medidos na silver (quarentena e painel de qualidade da Fase 2).

Alternativas avaliadas:
- *Schema declarado explicitamente:* exige alteração de código a cada mudança na origem. Será aplicado na silver, como contrato de dados.
- *Inferência de tipos (`inferColumnTypes = true`):* tipos corretos já na bronze, mas maior sensibilidade a valores fora do padrão e uma leitura adicional para inferência.

---

**Pergunta:** O que acontece quando a origem envia uma coluna nova?

**Resposta:** Será adotado o modo **`cloudFiles.schemaEvolutionMode = addNewColumns`** (padrão do Auto Loader). Ao encontrar um campo novo, o Auto Loader inclui a coluna no schema armazenado e interrompe o stream com `UnknownFieldException`. Ao ser reiniciado, o stream continua com o schema atualizado; os registros anteriores ficam com valor nulo na nova coluna.

A interrupção é intencional: sinaliza a mudança de schema. Para que a ingestão não dependa de intervenção manual, a função de ingestão fará uma nova tentativa automática quando receber essa exceção. Na Fase 7, o job terá retentativas configuradas, conforme recomendação da documentação oficial.

Alternativa avaliada: *modo `rescue`.* O stream nunca é interrompido e os campos novos são gravados apenas na coluna de dados resgatados. Descartado porque oculta a mudança de schema em vez de incorporá-la à tabela.

---

**Pergunta:** O que acontece com valores que não se encaixam no schema?

**Resposta:** São gravados na coluna **`_rescued_data`**, que o Auto Loader adiciona automaticamente. Ela armazena em JSON os valores que não puderam ser associados ao schema: campos com diferença de maiúsculas e minúsculas, tipos incompatíveis ou campos não previstos. Com todas as colunas como string, a ocorrência deve ser rara; registros com `_rescued_data` preenchido indicam problema na origem e serão monitorados pelo painel de qualidade.

#### B5. Colunas de auditoria

**Pergunta:** Como serão preenchidas as colunas de auditoria?

**Resposta:**

| Coluna | Origem do valor | Finalidade |
|---|---|---|
| `_ingested_at` | `current_timestamp()` | momento em que o registro foi gravado na bronze |
| `_source_file` | `_metadata.file_path` | arquivo de origem do registro |
| `_source_file_modified_at` | `_metadata.file_modification_time` | momento em que o arquivo foi gravado na landing; permite medir o atraso de ingestão |
| `_run_id` | parâmetro `run_id` do notebook | execução que gravou o registro |
| `sim_date` | extraída da pasta `sim_date=...` | dia simulado do arquivo |

Detalhes de implementação:
- `_metadata` é uma coluna oculta, disponível em toda leitura de arquivos, e precisa ser selecionada explicitamente (`F.col("_metadata.file_path")`). A função `input_file_name()` está obsoleta e não deve ser usada.
- Na execução por job (Fase 7), o `run_id` é recebido pela referência dinâmica `{{job.run_id}}`. Na execução interativa, o notebook gera um identificador (`uuid4`) quando o parâmetro não é informado.

Justificativa: essas colunas atendem ao requisito de rastreabilidade acordado com o negócio. Para qualquer registro, é possível identificar de onde veio, quando chegou e qual execução o gravou.

#### B6. Responsabilidade pela deduplicação

**Pergunta:** Quando a origem envia o mesmo dado em dois arquivos diferentes, a bronze deve deduplicar ou guardar os dois registros?

**Resposta:** **A bronze guarda os dois registros. A deduplicação é responsabilidade da silver.** São dois tipos de duplicidade, com tratamentos distintos:

| | Duplicidade técnica | Duplicidade de negócio |
|---|---|---|
| Causa | o nosso processo leu o mesmo arquivo mais de uma vez (reexecução, perda de checkpoint) | a origem enviou o mesmo dado mais de uma vez (reenvio, retentativa, defeito) |
| Exemplo | reprocessar `orders_2026-09-06_part-000.json` | receber `..._part-000.json` e `..._part-000_retry-1.json` |
| Tratamento | impedida na bronze: checkpoint e nomes determinísticos (A3 e B3) | removida na silver: deduplicação por chave de negócio |
| Pode existir na bronze? | não | sim, por ser um fato ocorrido na origem |

Justificativa: a bronze é append-only e fiel à origem. Preservar as duplicatas de negócio permite:
- **auditar** e comprovar o comportamento dos sistemas de origem;
- **medir** a taxa de reenvio por entidade no painel de qualidade;
- **reprocessar** a silver com outra regra de deduplicação, sem perda de informação.

A silver (Fase 2) fará a deduplicação de forma determinística: pela chave de negócio de cada entidade (por exemplo, `order_id` + `event_sequence` nos eventos de status), mantendo o registro de menor `_ingested_at`.

---

## 6. Resumo das decisões

| # | Decisão |
|---|---|
| A1 | Datas deslocadas em 3.654 dias (522 semanas); relógio diário com `dias_por_execucao` configurável; progresso em `ops.simulator_runs`; a data de referência do negócio é a data simulada |
| A2 | Pedidos divididos em cabeçalho (`orders`) e eventos (`order_status_events`); `customers` e `sellers` como feeds CDC; `products`, `geolocation` e `category_translation` em carga inicial |
| A3 | JSON Lines; caminho determinístico `raw_data/<entidade>/sim_date=AAAA-MM-DD/<entidade>_AAAA-MM-DD_part-000.json`; escrita em `_staging` seguida de move |
| A4 | Um arquivo por entidade por dia; 1 dia por execução na operação e 30 na carga histórica |
| A5 | Falhas determinísticas (semente por data, hash por chave), configuradas em `CHAOS_CONFIG` e registradas em `ops.simulator_chaos_log`; datas fixas em 2026-10-07, 2026-10-08 e 2026-10-10; cabeçalho do pedido e feeds CDC nunca atrasam |
| A6 | Mudanças cadastrais sintéticas de endereço (`UPDATE`) e exclusões (`DELETE`, só com as chaves) em `customers` e `sellers`, definidas por hash da chave |
| A7 | Reinício controlado da simulação com as falhas ativas, pelo notebook `setup/99_reset_simulacao`, com confirmação explícita |
| B1 | Bronze em notebook com Structured Streaming e Auto Loader, função genérica por entidade |
| B2 | `trigger(availableNow=True)`, sempre declarado |
| B3 | Checkpoint e schema em `/Volumes/olist/ops/checkpoints/bronze/<entidade>/` |
| B4 | Colunas como string na bronze; `addNewColumns` com nova tentativa automática; `_rescued_data` monitorada; tipagem na silver |
| B5 | `_ingested_at`, `_source_file`, `_source_file_modified_at`, `_run_id`, `sim_date` |
| B6 | Bronze preserva duplicatas de negócio; silver deduplica; duplicidade técnica impedida por checkpoint e nomes determinísticos |

## 7. Riscos e pontos de atenção

| Risco | Mitigação |
|---|---|
| Uso de `current_date()` em regras de negócio | regra de projeto: a referência de "hoje" é a data simulada em `ops.simulator_runs` |
| Exclusão acidental de checkpoint, gerando duplicidade na bronze | checkpoints isolados no schema `ops`; reprocessamento completo apenas como operação intencional e documentada |
| Esgotamento da cota de compute | carga histórica em lotes de 30 dias; trigger `availableNow` |
| Nome de arquivo não determinístico introduzido por engano | restrição registrada em A3 e verificada no teste de aceite de idempotência |
| Interrupção do stream por mudança de schema | nova tentativa automática na função de ingestão; retentativas no job (Fase 7) |

## 8. Plano de implementação e critérios de aceite

| Etapa | Entrega | Critério de aceite |
|---|---|---|
| 1.2 Simulador base | geração de dias sem falhas; `ops.simulator_runs`; deslocamento de datas; divisão de pedidos em cabeçalho e eventos | regerar um mesmo dia produz arquivos idênticos |
| 1.3 Bronze | Volume de checkpoints; função `ingest_entity`; colunas de auditoria; uma tabela por entidade | contagens da bronze conferem com os arquivos gerados |
| 1.4 Falhas e CDC | reinício controlado (A7); `CHAOS_CONFIG`; `ops.simulator_chaos_log`; feeds CDC de `customers` e `sellers` (A6) | falhas registradas no gabarito e reproduzíveis; mudanças cadastrais coerentes (nenhuma após a exclusão) |
| 1.5 Aceite | testes de idempotência e de evolução de schema; ADRs da fase | três execuções sem duplicidade técnica; stream íntegro após inclusão de coluna nova |
