# ADR 0001 — Organização da camada landing

- **Status:** aceito
- **Data:** 2026-11-02
- **Fase:** 0 — Fundação

## Contexto

O projeto usa o dataset público da Olist (9 CSVs, ~126 MB, 2016–2018) como base de um **simulador** que vai reencenar os pedidos "no presente", gravando lotes de arquivos aos poucos (com duplicatas, atrasos e eventos fora de ordem). A camada bronze vai ler esses arquivos com **Auto Loader**.

Na primeira versão do setup, os 9 CSVs ficaram soltos na raiz do volume `olist.landing.raw_data`, junto com uma pasta `.kagglehub_cache` contendo uma segunda cópia dos mesmos arquivos. Isso traz três problemas:

1. O Auto Loader trata tudo o que aparece numa pasta como arquivo novo a ingerir. Misturar o dataset original com os lotes do simulador geraria duplicidade na bronze.
2. Cada fluxo do Auto Loader deve ler uma pasta com um único schema; CSVs de entidades diferentes na mesma pasta inviabilizam isso.
3. O cache duplicava o armazenamento e era um "lixo" dentro da zona de ingestão.

## Decisão

Separar a landing em **dois volumes com papéis distintos**:

| Volume | Papel | Quem escreve | Quem lê |
|---|---|---|---|
| `olist.landing.olist_source` | Dataset original, intocado (Kaggle `olistbr/brazilian-ecommerce`, **versão 2 fixada**) | Notebook `setup/00_setup_olist` | Simulador |
| `olist.landing.raw_data` | Zona de chegada dos lotes | Simulador | Auto Loader (bronze) |

Regras complementares:

- `raw_data` terá **uma subpasta por entidade** (`orders/`, `order_items/`, `order_payments/`, `order_reviews/`, `customers/`, `sellers/`, `products/`, ...), cada uma com um único schema e lida por um fluxo próprio do Auto Loader. Eventos de status e feeds de mudança cadastral também terão subpastas próprias, definidas na Fase 1.
- O cache do `kagglehub` fica no disco local temporário do compute (`tempfile.mkdtemp()`) e é apagado ao final; nada além dos arquivos finais é gravado em Volumes.
- `olist_source` é tratado como **somente leitura** depois do setup.
- O setup é idempotente: `CREATE ... IF NOT EXISTS`, `COMMENT ON` e cópia com sobrescrita.

## Alternativas consideradas

1. **Um volume só, com pastas `source/` e `incoming/`.** Mais simples, mas o isolamento depende só de convenção de caminho; um Auto Loader apontado para a raiz leria tudo. Volumes separados deixam a fronteira explícita e permitem permissões diferentes no futuro (Fase 6).
2. **Ingerir o dataset original direto na bronze, sem simulador.** Mais rápido, mas elimina os desafios de produção que são objetivo do projeto (chegada incremental, duplicidade, atraso, fora de ordem).
3. **Particionar `raw_data` por data (`raw_data/2026-11-02/...`).** Bom para retenção, mas mistura entidades na mesma pasta. Pode ser combinado depois, *dentro* de cada entidade (`orders/2026-11-02/`), se for útil.
4. **Cache do `kagglehub` num volume dedicado.** Funcionaria, mas o disco local temporário se mostrou suficiente e não deixa resíduo.

## Consequências

- (+) A bronze nunca lê o dataset original por engano; reprocessar a bronze não exige baixar o dataset de novo.
- (+) Cada entidade tem schema próprio e evolução de schema isolada.
- (+) O simulador pode ser reexecutado a partir de uma fonte estável e versionada.
- (−) Mais um volume para gerenciar e documentar.
- (−) O download depende de acesso à internet a partir do serverless; se a Free Edition bloquear o domínio do Kaggle no futuro, o plano B é baixar manualmente e subir os CSVs para `olist_source` pela interface.
