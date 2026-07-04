# 01 · Regras de negócio

Estas regras valem para **todos** os cálculos e telas do desafio. Elas espelham as
regras do produto real, então precisão aqui conta muito na avaliação.

---

## 1. Registros excluídos (soft delete) — ⚠️ regra crítica

Toda resposta em `seed.json` tem um campo `deletedAt`. Quando ele **não é `null`**, a
resposta foi excluída logicamente (por exemplo, a pedido do titular — LGPD) e
**não pode entrar em nenhuma consulta, cálculo, listagem ou exportação**.

A mesma regra vale para o que **você** cria: o DELETE de contato é soft delete
(marque `deletedAt`, não apague a linha), e contatos excluídos somem de todas as
leituras.

> É a regra número 1 do nosso banco de produção: `WHERE deleted_at IS NULL` em toda
> query. Esquecer esse filtro é o erro clássico — e os valores de conferência abaixo
> denunciam na hora.

## 2. Classificação NPS (pesquisas tipo `NPS`, escala 0–10)

| Classe | Notas |
|---|---|
| **Detrator** | 0 a 6 |
| **Neutro** | 7 a 8 |
| **Promotor** | 9 a 10 |

## 3. Fórmulas

Com `total` = respostas válidas (não excluídas) no recorte filtrado:

```
promoters_pct  = promotores / total × 100
neutrals_pct   = neutros    / total × 100
detractors_pct = detratores / total × 100

nps_score      = promoters_pct − detractors_pct
```

- `responsesCount` = total de respostas válidas no recorte, **de qualquer tipo de
  pesquisa** (é a única métrica agnóstica de escala).
- `csatAvg` (pesquisas tipo `CSAT`, escala 1–5) = média aritmética das notas.

### Onde calcular e como arredondar

- **Agregações no banco**: contagens, somas, `GROUP BY` e `JOIN` acontecem em SQL.
  Formatar/arredondar o resultado no C# é aceitável — carregar as respostas para a
  memória e contar com LINQ, não.
- Calcule com precisão total e arredonde **apenas ao formatar**:

| Valor | Formato |
|---|---|
| `npsScore` | inteiro (ex.: `24`) — arredonde a diferença das porcentagens *cruas*, não das já arredondadas |
| percentuais | 1 casa decimal (ex.: `48.8`) |
| `csatAvg` | 2 casas decimais (ex.: `3.91`) |

## 4. Regras das consultas de analytics

A regra de escalas vale para o resumo **obrigatório**; as demais entram em cena nos
**bônus** de filtros e agrupamentos.

- Misturar escalas não faz sentido: métricas NPS consideram **apenas** respostas das
  pesquisas tipo `NPS`; `csatAvg` considera apenas as tipo `CSAT` (e `responsesCount`
  conta tudo — seção 3).
- **Período** (`from`/`to`, bônus): filtra por `respondedAt`, intervalo **inclusivo**
  nas duas pontas (o dia `to` inteiro conta). Formato `yyyy-MM-dd`.
- **Pesquisa** (`surveyId`, bônus): restringe às respostas daquela pesquisa.
- **Agrupamento mensal** (bônus): agrupe pelo mês de `respondedAt` **em UTC** (as
  datas do seed já estão em UTC, com horários longe da virada do dia — não há
  pegadinha de timezone). Rótulo do bucket: `"2026-01"`, `"2026-02"`, ...
- **Meses sem respostas não entram na série** — só gere bucket para mês com pelo menos
  uma resposta válida no recorte (NPS de zero respostas não existe; não divida por
  zero). Ex.: filtrando pela pesquisa "NPS Relacionamento 2026", a série tem só
  `2026-03` e `2026-06`.
- **Segmento** (bônus) vem do contato (`contacts.segment`) — a consulta por segmento
  exige JOIN entre respostas e contatos.

## 5. Validações do cadastro de contatos

| Campo | Regra |
|---|---|
| `name` | obrigatório, não vazio |
| `email` | obrigatório, formato válido, **único** entre contatos não excluídos (comparação case-insensitive) → duplicado responde `409` |
| `segment` | opcional; texto livre (o seed usa "Plano Black", "Plano Fit" e "Corporativo") |

Erros de validação respondem `400` (ou `409` para e-mail duplicado) com corpo JSON
`{ "error": "mensagem legível" }`.

## 6. Valores de conferência ✅

Use esta tabela para validar suas consultas SQL. Todos os valores consideram o período
completo do seed (01/01/2026 a 30/06/2026), **sem** filtro de pesquisa, já aplicada a
regra do `deletedAt`.

### Resumo geral

| Métrica | Valor |
|---|---|
| `responsesCount` (todas as pesquisas) | **1246** |
| Respostas válidas (NPS) | **978** |
| Promotores / Neutros / Detratores | **477 / 256 / 245** |
| `promoters_pct` | **48.8** |
| `neutrals_pct` | **26.2** |
| `detractors_pct` | **25.1** |
| `npsScore` | **24** |
| `csatAvg` (268 respostas CSAT) | **3.91** |

### NPS por mês *(bônus)*

| Mês | Respostas | NPS |
|---|---|---|
| 2026-01 | 104 | **17** |
| 2026-02 | 112 | **26** |
| 2026-03 | 270 | **11** |
| 2026-04 | 96 | **14** |
| 2026-05 | 121 | **22** |
| 2026-06 | 275 | **42** |

### NPS por segmento *(bônus — JOIN com contatos)*

| Segmento | Respostas NPS | NPS |
|---|---|---|
| Plano Black | 415 | **31** |
| Corporativo | 292 | **19** |
| Plano Fit | 271 | **18** |

### Com filtro de pesquisa *(bônus filtros)*

| Filtro | NPS |
|---|---|
| Só "NPS Pós-Treino" (id 1) | **21** |
| Só "NPS Relacionamento 2026" (id 2) | **29** |

Se algum número não bater, os suspeitos de sempre são: o filtro de `deletedAt`
(seção 1), a mistura de escalas NPS × CSAT (seção 4) e arredondamento antes da hora
(seção 3).
