# ADR 0012 — SLA do monitor e número oficial versus ao vivo

- **Status:** aceito
- **Data:** 2026-11-17
- **Fase:** 5 — Streaming

## Contexto

O escopo deixou em aberto o SLA do monitor quase em tempo real (5 ou 15 minutos). Além disso, com eventos tardios, o mesmo KPI pode ter dois valores: o da gold batch e o do estado ao vivo. Sem regra explícita, a Diretoria veria números diferentes sem explicação.

## Decisão

1. **SLA de projeto: 15 minutos**, medido entre a chegada do arquivo na landing e o aparecimento da linha em `gold_orders_live`. Reavaliar na Fase 8 com a duração real de cada ciclo.
2. **A gold batch (`fact_orders` e demais) continua sendo a fonte oficial dos KPIs.** `gold_orders_live` é a visão **operacional** ao vivo, para agir sobre pedidos abertos.
3. O **dicionário de métricas v1.2** registra essa distinção, e a documentação da tabela repete na descrição.
4. A validação confere a **equivalência de status** entre `gold_orders_live` e `fact_orders` para os pedidos até 2028-08-21.

## Alternativas consideradas

1. **SLA de 5 minutos.** Alerta mais fresco, mas muito mais execuções; cada uma paga a inicialização do serverless e consome a cota diária do Free Edition.
2. **Dois números oficiais.** Gera discussão sobre qual é "o certo"; descartado.
3. **Trocar a fonte oficial para o ao vivo.** Perderia a validação já feita (108 verificações da Fase 4) e a estabilidade dos KPIs.

## Consequências

- (+) Cabe na cota do Free Edition, com folga para inicialização e variação de duração.
- (+) Fica claro qual número vale em reunião e qual serve para operar.
- (−) O alerta pode ter até 15 minutos de atraso.
- (−) O Power BI e o Genie, se consumirem `gold_orders_live`, precisam rotular a tabela como operacional.
