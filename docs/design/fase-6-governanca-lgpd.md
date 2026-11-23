# Documento de Desenho Técnico — Fase 6: Governança e LGPD

| | |
|---|---|
| **Projeto** | Olist Lakehouse |
| **Fase** | 6 — Governança e LGPD |
| **Responsável técnico** | João Guilherme Araujo |
| **Data** | 2026-11-18 |
| **Status** | Aprovado em 2026-11-18, todas as decisões na opção recomendada; registradas nos ADRs [0016](../adr/0016-perfis-grupos-e-grants-por-camada.md) (G1, G2), [0017](../adr/0017-classificacao-de-pii-e-mascaras-de-coluna.md) (G3, G4, G8), [0018](../adr/0018-filtro-de-linha-regional.md) (G5) e [0019](../adr/0019-exclusao-de-cliente-e-evidencia.md) (G6, G7). Ajuste: o dono do pipeline passa a ser o grupo `olist_engenharia`, para a isenção das máscaras não depender de e-mail no código |
| **Documentos relacionados** | [Desenho da Fase 4 (gold)](fase-4-gold-metricas.md) · [Desenho da Fase 5 (streaming)](fase-5-streaming.md) · [ADR 0006](../adr/0006-historico-scd2-do-cadastro.md) · [ADR 0008](../adr/0008-scorecard-de-vendedores-pedidos-compartilhados-e-cadastro-na-compra.md) · [ADR 0015](../adr/0015-leitura-em-streaming-da-bronze-com-skipchangecommits.md) · [Dicionário de métricas](../dicionario_metricas.md) |

---

## 1. Objetivo deste documento

Registrar o desenho da governança **antes** da implementação: quem vê o quê, o que fica mascarado, como um cliente é excluído de ponta a ponta e que evidência prova isso. As decisões G1 a G8 estão escritas com as opções avaliadas e uma recomendação; a Fase 6 só implementa depois da aprovação.

## 2. Contexto e alinhamento com o negócio

Até a Fase 5 todo objeto é lido pelo dono do catálogo, sem separação por público. A empresa precisa mostrar que trata dado pessoal com controle.

| Área | Dor | O que a Fase 6 entrega |
|---|---|---|
| DPO (encarregado de dados) | não consegue provar quem acessou dado pessoal nem que uma exclusão foi feita | trilha de auditoria e evidência de exclusão por solicitação |
| Gestão e BI | precisam de números por região sem enxergar o cadastro dos clientes | perfis com máscaras por categoria de dado pessoal e filtro de linha por região |
| Atendimento (CX) | precisa contatar o cliente do pedido atrasado | a lista de risco não tem dado pessoal; o contato fica com o perfil que tem base legal |
| Diretoria | risco de exposição de dado pessoal em relatórios e no Power BI | dado pessoal protegido na origem, e não em cada relatório |

**Critério de aceite do épico E6** (Roadmap): demonstração com **dois perfis vendo dados diferentes** e **evidência de exclusão**.

## 3. Estado de partida (inventário em 2026-11-18)

| Fato | Valor |
|---|---|
| Tags de PII (`pii`, `pii_categoria`) | 25 colunas em 8 tabelas: silver `customers`, `sellers` e os dois históricos; gold `dim_customer`, `dim_seller`, `fact_orders`, `gold_seller_scorecard` |
| Categorias | `identificador` (só `customer_unique_id`) e `localizacao` (cidade, UF e prefixo de CEP de clientes e vendedores) |
| Tabelas com `contem_pii = false` | as demais da gold, `gold_orders_live`, `vw_pedidos_em_risco`, as de `ops` da Fase 5 e `quality.validacao_live` |
| **Lacunas de classificação** | **bronze** (`customers`, `sellers`, `order_reviews`) sem tag; **silver `order_reviews`** com `review_comment_message` e `review_comment_title` (texto livre, pode ter nome, telefone, e-mail) sem tag; **quarentenas** `customers_quarentena` (cadastro completo) e `order_reviews_quarentena` sem tag; `_corrupt_record` das quarentenas guarda a linha bruta |
| Cliente x pessoa | 99.291 `customer_id` (um por pedido) para 95.695 `customer_unique_id` (pessoas): excluir uma pessoa remove várias linhas |
| Dono dos objetos | o próprio usuário do projeto, em todos os schemas; nenhum `GRANT` existe (o catálogo não tem permissões concedidas) |
| Grupos | o workspace já tem grupos criados para outras finalidades; **não serão reaproveitados**. Só há 1 usuário humano |
| Funções disponíveis | `CREATE FUNCTION` funciona em `olist.ops`; `is_member()` enxerga os grupos do workspace |
| `system.access` | `table_lineage` (26 mil linhas) e `audit` (54 mil linhas no último dia) acessíveis: serve de trilha e de lineage |
| Secrets | nenhum escopo criado |
| CDF e deletion vectors | silver (AUTO CDC) com Change Data Feed ligado; as tabelas têm deletion vectors: o `DELETE` só marca a linha, e a remoção física exige `REORG ... APPLY (PURGE)` antes do `VACUUM` |
| Máscara em materialized view | existe `ALTER MATERIALIZED VIEW ... SET MASK`, mas a identidade que roda o pipeline precisa estar isenta da máscara, senão o dado gravado já nasce mascarado ([requisitos](https://docs.databricks.com/gcp/en/data-governance/unity-catalog/abac/requirements)) |

## 4. Decisões a tomar

### G1 — Perfis e grupos

**Pergunta:** que perfis existem e como são representados?

| Opção | Prós | Contras |
|---|---|---|
| Reaproveitar grupos que já existem no workspace | nada a criar | foram criados para outras finalidades; mistura permissões |
| **Grupos novos `olist_analista`, `olist_gestor`, `olist_dpo` (recomendada)** | nome explica o papel; permissões isoladas do projeto | exige criar grupos (peço sua autorização na 6.2) |

**Recomendação:** 3 perfis de consumo, mais o dono do pipeline (engenharia, isento de máscaras, porque as materialized views da gold são gravadas por ele).

| Perfil | Quem é | Vê |
|---|---|---|
| `olist_analista` | analista de BI | gold sem dado pessoal direto: hash no lugar do identificador, UF sem cidade nem CEP |
| `olist_gestor` | gestor de área | o do analista, mais cidade e CEP truncado, com filtro de linha por região |
| `olist_dpo` | encarregado de dados | tudo em claro, mais a trilha de auditoria; é quem executa a exclusão |
| engenharia (dono) | João | tudo; roda os pipelines |

Como só há 1 usuário, a **demonstração dos dois perfis** é feita alternando a participação nos grupos (entra e sai de `olist_dpo`), consultando a mesma tabela e guardando o resultado como evidência.

### G2 — Quem acessa qual camada (`GRANT`)

| Camada | analista | gestor | DPO |
|---|---|---|---|
| `landing`, `bronze`, `silver` | sem acesso | sem acesso | leitura (precisa localizar o titular) |
| `gold` (tabelas e Metric Views) | leitura | leitura | leitura |
| `quality`, `ops` | sem acesso, exceto leitura de `quality.validacao_*` | idem | leitura |

**Recomendação:** acesso por camada, o menor possível: quem consome vê só a gold. É o que impede o atalho de ler o cadastro na silver. As quarentenas ficam no DPO porque guardam o registro bruto. Cada perfil precisa de `USE CATALOG` e `USE SCHEMA` além do `SELECT`.

### G3 — Máscaras de coluna por categoria

**Pergunta:** o que cada perfil enxerga em cada coluna marcada como PII?

| Coluna (categoria) | analista | gestor | DPO |
|---|---|---|---|
| `customer_unique_id` (identificador) | hash com salt | hash com salt | em claro |
| `customer_state` (localização) | em claro | em claro | em claro |
| `customer_city` | `NULL` | em claro | em claro |
| `customer_zip_code_prefix` | `NULL` | 3 primeiros dígitos | em claro |
| `seller_*` (localização) | em claro (ver G4) | em claro | em claro |
| `review_comment_message` e `_title` (texto livre) | `NULL` | `NULL` | em claro |

**Por que o hash e não `NULL` no identificador:** o hash é estável, então o analista ainda conta clientes distintos e acompanha recompra (KPIs do dicionário), sem saber quem é a pessoa. **Por que a UF fica aberta:** o BI e o Genie analisam por estado e a UF sozinha não identifica ninguém.

**Mecanismo:** função SQL de máscara por categoria (`ops.mask_identificador`, `ops.mask_localizacao_*`, `ops.mask_texto_livre`) que usa a participação no grupo, aplicada com `ALTER ... ALTER COLUMN ... SET MASK`, guiada pelas tags `pii_categoria` (as colunas a mascarar saem do catálogo, não de uma lista escrita à mão). Alternativa descartada: *views seguras* (`vw_dim_customer_mascarada`): funcionam, mas o dado cru continua legível por quem receber `SELECT` na tabela de base, e cada objeto novo exige uma view nova.

**Pontos que o spike da 6.2 precisa comprovar:** (a) máscara em materialized view da gold sem mascarar a gravação pelo pipeline; (b) o salt do hash: escopo de secret (`secret()`) se o Free Edition permitir, senão salt em tabela restrita ao DPO, registrado como limitação; (c) máscaras e Metric Views, dashboard e Genie (herdam o perfil de quem consulta); (d) efeito colateral conhecido: *time travel* falha em tabela com máscara ativa.

### G4 — Localização do vendedor

**Pergunta:** cidade, UF e CEP do vendedor são dado pessoal?

A Fase 4 marcou como `localizacao`. Vendedor é, em geral, empresa: a LGPD protege a **pessoa natural**, e o endereço comercial é público. Mas o dataset não distingue empresa de microempreendedor individual.

| Opção | Prós | Contras |
|---|---|---|
| Mascarar como o cliente | conservador | quebra o scorecard por região e não protege ninguém de fato |
| **Não mascarar; trocar a categoria para `localizacao_comercial` (recomendada)** | decisão explícita e documentada; o scorecard segue útil | risco residual para vendedor pessoa física, registrado no ADR |

### G5 — Filtro de linha

**Pergunta:** qual o recorte de linhas por perfil?

**Recomendação:** o gestor vê só os pedidos das **UFs da sua região**, por uma tabela de mapeamento `olist.ops.acesso_regional` (grupo → UFs) lida pela função do filtro (`ALTER TABLE ... SET ROW FILTER`), aplicado em `fact_orders` e `dim_customer`. O analista e o DPO veem todas as linhas. O filtro vive numa tabela de regras, e não no código, para mudar o recorte sem alterar a função. Limites: filtros em materialized views seguem o mesmo cuidado de isenção do dono (G3); **um filtro por tabela**; os agregados que já somam todas as UFs (`gold_daily_kpis`) não são filtrados, porque não têm dado pessoal e o dicionário define o número oficial como o total.

### G6 — Exclusão de cliente ponta a ponta

**Pergunta:** o que "excluir um cliente" significa, dado que pedido é registro financeiro?

| Opção | Prós | Contras |
|---|---|---|
| Apagar também os pedidos | exclusão total | muda os KPIs oficiais e apaga registro com obrigação de guarda |
| **Apagar o cadastro e desvincular o pedido (recomendada)** | remove a pessoa; os KPIs não mudam; o pedido segue como registro anônimo | exige explicar a base legal no ADR |

**Escopo e ordem do procedimento** (notebook com parâmetro `customer_unique_id`, modo `simulacao` que só conta e modo `executar`):

1. localizar todos os `customer_id` da pessoa e gravar a solicitação em `ops.exclusoes_lgpd` (id, data, hash do titular, status) **sem guardar o dado pessoal em claro**;
2. `DELETE` físico em `bronze.customers`, `silver.customers`, `silver.customers_historico` (o `DELETE` do SCD2 só fecha a versão, ADR 0006; aqui as versões são **removidas**) e nas quarentenas (`customers_quarentena`, e a linha bruta em `_corrupt_record` quando houver);
3. atualizar a gold: `dim_customer` e `fact_orders.customer_unique_id` deixam de ter o titular ao reprocessar (o pedido permanece, sem vínculo com a pessoa);
4. `REORG TABLE ... APPLY (PURGE)` e `VACUUM` com retenção reduzida nas tabelas tocadas, para sumir dos arquivos físicos, e não só da leitura. O Change Data Feed guarda a linha apagada nos arquivos de mudança, então a purga e o `VACUUM` também cobrem esse histórico;
5. **estado ao vivo:** como o pedido permanece, `gold_orders_live` não precisa mudar (não tem dado pessoal). Isso resolve a pendência do ADR 0015. Se um dia a exclusão apagar o pedido, a rota é reconstruir o estado com checkpoint novo;
6. verificação independente: contar o titular em **todas** as tabelas (tem que dar 0), checar que os KPIs oficiais não mudaram e que uma leitura em versão antiga (*time travel*) já não devolve a pessoa.

**Alternativa descartada:** propagar a exclusão por Change Data Feed da bronze até a silver (ADR 0015, alternativa 2): exigiria ligar o feed na bronze e tratar `delete` em cada camada, e o procedimento explícito dá mais evidência por camada.

### G7 — Evidência e auditoria

**Pergunta:** o que prova, para o DPO, que a exclusão e os acessos aconteceram?

**Recomendação:** três fontes combinadas.

| Fonte | Prova o quê |
|---|---|
| `ops.exclusoes_lgpd` (nova) | quem pediu, quando, quantas linhas saíram de cada tabela, versão Delta antes e depois, resultado da verificação |
| `DESCRIBE HISTORY` das tabelas | a operação `DELETE`/`PURGE`/`VACUUM` e suas métricas |
| `system.access.audit` e `system.access.table_lineage` | quem consultou dado pessoal e por quais caminhos o dado flui (lineage das 8 tabelas com PII) |

A evidência é gravada em tabela (`quality.validacao_governanca`) e exportada num relatório do DPO, sem dado pessoal em claro.

### G8 — Fechar as lacunas de classificação

**Recomendação:** antes das máscaras, **classificar o que falta**: tags `pii` e `pii_categoria` em `bronze.customers`, `bronze.sellers`, `bronze.order_reviews` (texto livre, categoria nova `texto_livre`), `silver.order_reviews`, nas duas quarentenas e em `fact_reviews` se houver cópia do texto (hoje não há). A validação da fase passa a **falhar** se existir coluna com nome de dado pessoal sem tag, para a lacuna não voltar.

## 5. Etapas propostas

| Etapa | Entrega | Branch sugerida |
|---|---|---|
| 6.1 | Este desenho, ADRs 0016 a 0019 e a pergunta dos grupos | `fase-6.1/desenho-governanca` |
| 6.2 | Grupos, `GRANT` por camada (G1, G2), classificação (G8) e máscaras (G3, G4) | `fase-6.2/perfis-e-mascaras` |
| 6.3 | Filtro de linha regional (G5) | `fase-6.3/filtros-de-linha` |
| 6.4 | Exclusão de cliente ponta a ponta com evidência (G6, G7) | `fase-6.4/exclusao-de-cliente` |
| 6.5 | Auditoria e lineage, `quality.validacao_governanca`, documentação, dicionário, encerramento | `fase-6.5/auditoria-e-encerramento` |

## 6. Como será validado

| Critério | Como se prova |
|---|---|
| Dois perfis veem dados diferentes | mesma consulta como analista, gestor e DPO; resultado gravado na tabela de validação |
| Nenhuma coluna de PII sem máscara | cruzar `information_schema.column_tags` com as máscaras aplicadas (0 colunas sem máscara) |
| Máscara não quebra o pipeline | atualização da gold depois das máscaras termina sem erro e com as mesmas contagens |
| KPIs oficiais inalterados | os 8 KPIs antes e depois das máscaras e da exclusão, iguais |
| Exclusão completa | titular com 0 linhas em todas as tabelas, inclusive quarentenas e históricos, e ausente em *time travel* depois do `VACUUM` |
| Exclusão repetível | rodar duas vezes dá o mesmo resultado, sem erro |
| Trilha | solicitação registrada e operações visíveis em `DESCRIBE HISTORY` e `system.access.audit` |

## 7. Riscos e limitações do Free Edition

| Risco | Plano |
|---|---|
| Grupos só no nível do workspace, sem console de conta | testar `is_member()` na 6.2; se necessário, a função lê uma tabela de associação, registrado em ADR |
| Sem escopo de secrets para o salt | salt em tabela restrita ao DPO |
| Máscara em materialized view pode mascarar a própria gravação | spike da 6.2 antes de aplicar em larga escala; plano B: views seguras para as dimensões |
| Retenção do `VACUUM` mínima exige desligar a verificação de segurança | só dentro do procedimento de exclusão, com registro |
| Power BI importa as tabelas com a identidade de quem publica | relatórios por público e consumo pela gold mascarada; a segurança por linha do Power BI não substitui a do Unity Catalog |

## 8. Fora do escopo

Criptografia em nível de coluna, *tokenização*, retenção automática de dados (ciclo de vida), consentimento do titular e gestão de incidentes. A Fase 8 revisa o custo do filtro de linha e das máscaras na consulta.

## 9. Perguntas aprovadas em 2026-11-18

1. Autoriza criar os grupos `olist_analista`, `olist_gestor` e `olist_dpo` na 6.2?
2. G3: a matriz de máscaras serve, em especial hash no identificador e `NULL` na cidade do analista?
3. G4: manter o vendedor sem máscara, com a categoria `localizacao_comercial`?
4. G6: apagar o cadastro e manter o pedido anônimo (e não apagar o pedido)?
