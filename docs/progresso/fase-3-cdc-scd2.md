# Fase 3 — CDC e SCD2

- **Período previsto:** a definir na abertura da fase
- **Status:** a iniciar — começa pelo contexto de negócio e pelo desenho técnico, aprovado antes da implementação
- **Branch/PRs:** —

## Objetivo

**Negócio:** saber o endereço e o cadastro de clientes e vendedores **na data de cada pedido**, e não só o atual, para que as métricas históricas não mudem quando alguém muda de cidade ou é excluído.

**Técnico:** dimensões com histórico (SCD tipo 2) construídas a partir do feed de mudanças (`operation`, `change_sequence`, `change_ts`) de `customers` e `sellers`, com `create_auto_cdc_flow` e tratamento explícito das exclusões.

O contexto de negócio, o escopo e os critérios de aceite serão detalhados na abertura da fase.

## Pendências herdadas da Fase 2

Ver "Encerramento" em [fase-2-silver-qualidade.md](fase-2-silver-qualidade.md), em especial:

- a silver SCD1 só guarda a última versão; o histórico vem do feed;
- 288 clientes e 21 vendedores têm `DELETE` como última versão, e a linha `DELETE` chega sem atributos;
- `change_ts` e `sim_date` do cadastro chegam a 2028.

## Etapas

_(a definir no desenho da fase)_

## O que foi feito

_(preencher durante a fase)_

## Decisões

_(preencher durante a fase)_

## Problemas encontrados e como foram resolvidos

_(preencher durante a fase)_

## Aprendizados

_(preencher durante a fase)_

## Encerramento

_(preencher ao concluir)_
