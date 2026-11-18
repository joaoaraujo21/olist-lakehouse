# ADR 0009 — Simulador em modo stream e relógio fino

- **Status:** aceito
- **Data:** 2026-11-17
- **Fase:** 5 — Streaming

## Contexto

O monitor da Fase 5 precisa medir latência em minutos, mas o simulador tem relógio **diário** (uma pasta `sim_date=` por dia). A simulação está gerada até 2028-08-21 e cerca de 2 meses do dataset ainda não foram emitidos. O risco de atraso também depende de uma noção de "hoje" que não pode ser a data real: o dataset está deslocado no tempo.

## Decisão

1. **Novo modo `stream`** no simulador, usado só para os últimos 2 meses. Cada execução emite uma **fatia de horas simuladas** (proposta inicial: 6 h), com um arquivo por fatia. O modo `batch` atual não muda.
2. **Falhas mantidas:** reenvio de arquivo, registro duplicado, atraso de chegada (até 3 dias) e fora de ordem continuam sendo injetados, e o gabarito `ops.simulator_chaos_log` passa a registrar também a fatia.
3. **Nome da pasta** passa a conter a data e a fatia; a landing continua organizada por entidade (ADR 0001).
4. **"Hoje" é a data simulada corrente**, isto é, o maior `event_ts` já emitido pelo simulador, lido do estado dele. Nunca `current_date()`.

## Alternativas consideradas

1. **Manter o dia como unidade** e processar dia a dia. Não mexe no simulador, mas não permite medir latência dentro do dia.
2. **Gerar tudo de uma vez e fingir a chegada espaçada na leitura.** Não exercita a chegada real de arquivos, o checkpoint nem o Auto Loader.
3. **Reescrever o simulador com relógio fino para todo o período.** Custo alto e invalidaria as Fases 1 a 4.

## Consequências

- (+) O modo `batch` e tudo o que foi validado nas Fases 1 a 4 ficam intactos.
- (+) Há instantes distintos dentro do dia, então a latência pode ser medida de verdade.
- (+) O risco de atraso é reproduzível: depende só dos dados, não do dia em que roda.
- (−) O simulador passa a ter dois modos e a pasta de landing, dois formatos de nome; a ingestão da bronze precisa aceitar ambos.
- (−) Os 2 meses finais mudam retroativamente alguns KPIs (por exemplo, cancelamento tardio); a validação da Fase 4 precisa distinguir "dados até 2028-08-21" de "dados completos".
