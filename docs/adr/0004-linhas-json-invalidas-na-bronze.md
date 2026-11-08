# ADR 0004 — Tratamento de linhas de JSON inválido na bronze

- **Status:** aceito
- **Data:** 2026-11-08
- **Fase:** 1 — Simulador e Bronze

## Contexto

A bronze lê os arquivos da landing com Auto Loader e guarda o dado como recebido. Para valores que não se encaixam no schema, o desenho previa a coluna `_rescued_data` (decisão B4), e o desenho A5 esperava que uma linha de JSON inválido também fosse resgatada nela.

A etapa 1.4 injetou uma linha de JSON inválido em `order_items` de 2026-10-10 e mostrou um comportamento diferente: a linha foi gravada como um registro com todas as colunas de negócio nulas e `_rescued_data` nulo. Somente `sim_date` (lida do nome da pasta) e as colunas de auditoria ficaram preenchidas. A documentação do Auto Loader confirma que `_rescued_data` captura apenas colunas desconhecidas, tipos divergentes e diferenças de maiúsculas; registros malformados não fazem parte desse mecanismo.

Na prática, o conteúdo original da linha não ficava na bronze, e o registro só seria percebido na silver de forma indireta, por uma regra de chave não nula.

## Decisão

Ativar a coluna de registros corrompidos do leitor JSON em todas as entidades da bronze:

- `columnNameOfCorruptRecord = "_corrupt_record"`;
- `cloudFiles.schemaHints = "_corrupt_record string"`, para que a coluna faça parte do schema inferido.

A linha que não puder ser lida como JSON é gravada com o texto original em `_corrupt_record` e as demais colunas nulas. A coluna é documentada como coluna técnica, como `_rescued_data`. Na silver (Fase 2), uma expectation `_corrupt_record IS NULL` envia esses registros para a quarentena.

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| (a) Manter o comportamento padrão | o texto original não fica na bronze, e a detecção na silver depende de um efeito colateral (chave nula), e não de uma regra explícita |
| (b) `badRecordsPath`, desviando as linhas inválidas para arquivos em uma pasta | os registros saem da tabela: a validação landing = bronze deixa de valer, e os arquivos desviados ficam fora da tabela Delta, sem colunas de auditoria e sem a governança da tabela. A documentação da Databricks desaconselha a opção, por uma condição de corrida que pode deixar passar registros corrompidos |

## Consequências

- A bronze passa a ter a coluna `_corrupt_record` em todas as tabelas, nula em condições normais.
- A contagem de linhas da landing continua igual à da bronze, e cada linha inválida fica auditável na própria tabela, com `_source_file`.
- A mudança altera o schema inferido das tabelas existentes. Ela foi aplicada com o reinício controlado da simulação (ADR 0003), e não sobre as tabelas já criadas.
- A silver deve tratar `_corrupt_record` junto com `_rescued_data` nas regras de qualidade da Fase 2.

## Referências

- Auto Loader best practices — registros corrompidos e `columnNameOfCorruptRecord`.
- Configure schema inference and evolution in Auto Loader — conteúdo do `_rescued_data`.
