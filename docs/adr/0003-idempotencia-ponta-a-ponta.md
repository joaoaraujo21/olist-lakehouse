# ADR 0003 — Idempotência ponta a ponta da ingestão

- **Status:** aceito
- **Data:** 2026-11-08
- **Fase:** 1 — Simulador e Bronze

## Contexto

A ingestão vai do simulador à bronze em três passos: o simulador grava arquivos diários na landing (`raw_data/<entidade>/sim_date=AAAA-MM-DD/`), o Auto Loader os lê de forma incremental e a bronze os acumula em tabelas Delta append-only. Qualquer um desses passos pode ser executado de novo: reexecução de um job que falhou, reprocessamento de um dia (`sim_date_override`) ou reinício da ingestão.

O requisito é que nenhuma reexecução produza **duplicidade técnica** (o mesmo dado gravado duas vezes pelo nosso processo). A **duplicidade de negócio** (a origem enviando o mesmo dado duas vezes, como um arquivo reenviado) deve chegar à bronze, porque é um fato da origem e é tratada na silver.

## Decisão

A idempotência é garantida pela combinação de quatro mecanismos. Nenhum deles é suficiente isoladamente.

1. **Nome de arquivo determinístico.** Cada entidade e dia simulado gera sempre o mesmo caminho (`<entidade>_<dia>_part-000.json`), sem UUID, data de execução ou qualquer valor variável. O reenvio simulado usa um nome próprio e igualmente fixo (`..._part-000_retry-1.json`).
2. **Conteúdo determinístico.** As linhas são ordenadas por chave (ou pelo hash da chave, nos arquivos embaralhados), e todas as falhas injetadas são decididas por hash da chave (`xxhash64`) ou por gerador com semente em texto. Gerar o mesmo dia em momentos diferentes produz arquivos idênticos.
3. **Controle de arquivos do Auto Loader.** O checkpoint de cada entidade registra os arquivos já processados e, com `cloudFiles.allowOverwrites = false`, um arquivo regravado no mesmo caminho não é lido novamente. Checkpoint e tabela são tratados como uma unidade: só são apagados juntos, pelo procedimento de reinício controlado (`setup/99_reset_simulacao`).
4. **Registros de controle idempotentes.** A execução do simulador é registrada com `MERGE` por `run_id` em `ops.simulator_runs`, e o gabarito das falhas é gravado com `replaceWhere` pelos dias da execução em `ops.simulator_chaos_log`. Reprocessar um dia substitui o gabarito desse dia, sem acumular linhas.

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| UUID ou horário no nome do arquivo, com deduplicação na bronze | cada execução geraria arquivos novos para o Auto Loader; a bronze precisaria distinguir duplicidade técnica de duplicidade de negócio, o que não é possível sem perder a segunda |
| `MERGE` por chave na bronze | descaracteriza a bronze como registro do que foi recebido: eliminaria as duplicidades de negócio e os registros atrasados, que a silver precisa observar |
| Sobrescrever a bronze por dia (`replaceWhere` por `sim_date`) | perde o histórico de recebimento e não trata registros atrasados, que chegam em arquivos de dias posteriores |
| `cloudFiles.allowOverwrites = true` | todo arquivo regravado seria lido de novo, e cada reprocessamento de um dia duplicaria os dados desse dia na bronze |

## Consequências

- A bronze recebe, de propósito, as duplicidades de negócio (registros repetidos e arquivos reenviados). A deduplicação é responsabilidade da silver, validada contra o gabarito.
- **Alterar a lógica do simulador exige o reinício controlado.** Um dia regerado com conteúdo diferente e o mesmo nome não é lido novamente pelo Auto Loader, e a landing e a bronze ficariam divergentes. O reinício apaga tabelas, checkpoints, landing e registros de controle, nessa ordem.
- Apagar apenas o checkpoint ou apenas a tabela de uma entidade é proibido fora do reinício: checkpoint sem tabela perde dados, e tabela sem checkpoint duplica.
- A garantia vale para reexecuções do nosso processo. Uma origem real que regrave arquivos com o mesmo nome e conteúdo novo exigiria outra estratégia (por exemplo, `allowOverwrites` com deduplicação na bronze), a ser avaliada se esse cenário surgir.

## Evidências (etapa 1.4)

- Duas cargas (2026-09-06 a 2026-10-31) com 1.039.181 linhas na landing e na bronze, sem divergência por entidade e dia.
- Reexecução da ingestão com 0 linhas novas em todas as entidades.
- Gabarito conferido contra a bronze: duplicatas e reenvios presentes na bronze exatamente como registrados.
