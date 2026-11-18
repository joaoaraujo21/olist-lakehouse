# ADR 0014 — Caminho rápido da bronze: ingestão seletiva por ciclo

- **Status:** aceito
- **Data:** 2026-11-18
- **Fase:** 5 — Streaming

## Contexto

O SLA do monitor é de 15 minutos entre a chegada do arquivo e a linha em `gold_orders_live` (ADR 0012). Na medição da etapa 5.2, um ciclo de uma fatia levou 8 a 9 minutos, dos quais cerca de 4,5 na bronze: o notebook processa as 10 entidades em série (cerca de 28 s cada, a maior parte em partida e checagem do Auto Loader), mesmo as que não têm arquivo novo. O monitor só usa duas: `order_status_events` (o estado) e `orders` (a data estimada de entrega).

## Decisão

1. O notebook da bronze ganha o parâmetro `entidades` (lista separada por vírgula; **vazio = as 10**, comportamento de sempre). Nome desconhecido interrompe a execução.
2. O ciclo de streaming ingere só `order_status_events` e `orders`. As demais entidades continuam sendo ingeridas por uma execução completa (sem o parâmetro), que lê os arquivos acumulados: os checkpoints são por entidade, então nada se perde nem se duplica.
3. O ciclo (simulador, bronze seletiva, streaming gold) fica como job de ciclo sob demanda, sem agendamento; a orquestração recorrente é da Fase 7.

## Alternativas consideradas

1. **Manter as 10 entidades por ciclo.** Mais simples, mas gasta cerca de 3 minutos a mais por ciclo e consome cota do Free Edition sem benefício para o monitor.
2. **Um notebook ou job de bronze por entidade.** Mais flexível, porém multiplica objetos para manter; o parâmetro entrega o mesmo efeito.
3. **Auto Loader com notificação de arquivos.** Reduziria a espera, mas depende de recursos de nuvem que o Free Edition não oferece.

## Consequências

- (+) A etapa da bronze caiu de cerca de 4,5 minutos para cerca de 1,7 (100 s); o ciclo completo passou de 8 a 9 para cerca de 7 minutos, e do arquivo até a linha em `gold_orders_live` leva cerca de 4 a 4,5 minutos, dentro do SLA.
- (+) O padrão do parâmetro não muda o comportamento das Fases 1 a 4.
- (−) Entre uma execução completa e outra, as tabelas bronze das demais entidades ficam defasadas. Quem lê essas tabelas (pipelines silver e gold) precisa de uma execução completa da bronze antes.
- (−) O `MERGE` de cada micro-lote ainda leva de 1,5 a 2 minutos com poucas dezenas de eventos, porque lê a tabela de estado e `bronze.orders` inteiras; é candidato à otimização da Fase 8.
