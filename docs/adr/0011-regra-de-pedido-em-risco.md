# ADR 0011 — Regra de pedido em risco de atraso

- **Status:** aceito
- **Data:** 2026-11-17
- **Fase:** 5 — Streaming

## Contexto

A Logística quer agir antes da entrega, enquanto o pedido ainda pode chegar a tempo. Hoje o atraso só é conhecido depois (`% no prazo`, ADR 0002). Na data simulada atual há 4.035 pedidos abertos (shipped, approved, invoiced, processing e created).

## Decisão

A regra vale **só para pedidos abertos** (nem `delivered`, nem `canceled`, nem `unavailable`), em dias da **data simulada** (ADR 0009):

| Nível | Regra |
|---|---|
| `atrasado` | a data estimada de entrega já passou |
| `em_risco` | faltam **até 2 dias** para a data estimada e o pedido ainda não foi despachado (`shipped`) |
| `normal` | demais |

- O valor **N = 2 dias** e os níveis ficam em **configuração**, não no código.
- A refinação por lead time mínimo observado (pedido já despachado, mas com prazo menor que o menor lead time) **fica para uma versão seguinte**.
- A coluna `risco_atraso` de `gold_orders_live` é calculada contra a data simulada corrente.

## Alternativas consideradas

1. **Incluir já o lead time mínimo.** Mais preciso, mas exige estudo de distribuição por rota e atrasa a fase; entra depois se o João pedir.
2. **N maior (3 ou 5 dias).** Lista maior e menos acionável para a Logística.
3. **Modelo estatístico de previsão de atraso.** É escopo da Fase 9 (ML, opcional).

## Consequências

- (+) Regra simples, explicável e verificável por um cálculo independente em SQL (critério de aceite 6).
- (+) Ajustar N não exige alterar código.
- (−) Um pedido despachado e com prazo apertado só vira `atrasado` depois de vencer; não é sinalizado como `em_risco`. É a limitação conhecida desta versão.
- (−) O resultado depende da data simulada; se ela não avançar, os níveis não mudam.
