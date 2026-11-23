# ADR 0017 — Classificação de PII e máscaras de coluna por categoria

- **Status:** aceito
- **Data:** 2026-11-18
- **Fase:** 6 — Governança e LGPD

## Contexto

As Fases 2 a 4 marcaram 25 colunas com `pii` e `pii_categoria` (`identificador`, `localizacao`). O inventário da Fase 6 achou dado pessoal **sem tag**: `bronze.customers`, `bronze.sellers`, `bronze.order_reviews`, `silver.order_reviews` (texto livre dos comentários), as quarentenas de clientes e de avaliações e o `_corrupt_record`/`_rescued_data` com a linha bruta. Falta também decidir o que cada perfil enxerga.

## Decisão

1. **Classificar primeiro.** As colunas sem tag recebem `pii = true` e a categoria pelo nome da coluna: `customer_unique_id` → `identificador`; `customer_city`, `customer_state`, `customer_zip_code_prefix` → `localizacao`; `review_comment_message` e `review_comment_title` → `texto_livre`; `_corrupt_record` e `_rescued_data` das entidades com dado pessoal → `registro_bruto`.
2. **Vendedor:** cidade, UF e CEP do cadastro do vendedor passam de `localizacao` para **`localizacao_comercial`** e **não são mascarados**. Vendedor é, em geral, empresa e o endereço comercial é público; o risco residual para vendedor pessoa física fica registrado aqui.
3. **Matriz de máscaras** (aplicada à gold, única camada que analista e gestor leem):

| Coluna | analista | gestor | DPO e engenharia |
|---|---|---|---|
| `customer_unique_id` | hash com salt | hash com salt | em claro |
| `customer_state` | em claro | em claro | em claro |
| `customer_city` | `NULL` | em claro | em claro |
| `customer_zip_code_prefix` | `NULL` | 3 primeiros dígitos | em claro |

4. O **hash é estável**: o mesmo cliente gera o mesmo valor, então contar clientes distintos e medir recompra continua possível sem identificar a pessoa. A UF fica aberta porque o BI e o Genie analisam por estado e ela sozinha não identifica ninguém.
5. As colunas a mascarar saem das **tags do catálogo** (`information_schema.column_tags`), e não de uma lista escrita à mão; a máscara é escolhida pela categoria e pelo nome da coluna.
6. **Salt:** guardado em `ops.governanca_config` (sem acesso para consumidores) e embutido no corpo da função de máscara; gerado uma vez e reaproveitado nas execuções seguintes. O Free Edition não tem escopo de secrets, e `secret()` dentro da máscara falharia para quem não tem permissão no escopo.
7. As funções ficam em `olist.ops` (`mask_identificador`, `mask_cidade`, `mask_cep`). Texto livre e registro bruto têm categoria, mas não máscara na 6.2: as colunas estão em camadas que o consumidor não lê (ADR 0016).
8. A validação da fase **reprova** se existir coluna com nome de dado pessoal sem tag, para a lacuna não voltar.

## Alternativas consideradas

1. **`NULL` no identificador.** Mais simples, mas o analista perderia a contagem de clientes distintos.
2. **Views seguras no lugar de máscaras.** Funcionam, mas o dado cru continua legível por quem receber `SELECT` na tabela de base e cada objeto novo exige uma view nova. Fica como plano B se a máscara em materialized view se mostrar instável.
3. **Mascarar também a silver e a bronze.** Defesa em profundidade, porém mexe em tabelas gravadas por pipeline sem ganho, já que ninguém além do DPO lê essas camadas.

## Consequências

- (+) Máscara guiada pela classificação: coluna nova com tag passa a ser tratada na próxima execução.
- (+) O analista mantém as métricas por cliente distinto e por UF.
- (−) *Time travel* falha em tabela com máscara ativa (limitação do Unity Catalog).
- (−) Quem sai de `olist_engenharia` e roda o pipeline grava dado mascarado na materialized view: a demonstração de perfis exige voltar ao grupo antes de qualquer atualização.
- (−) Trocar o salt muda todos os hashes; só se rotaciona de propósito.
