# 02 · Contrato da API REST

Este documento é o **contrato** entre o frontend e o backend — implemente exatamente
estes shapes e status codes. O que o contrato **não** define (schema do banco, nomes
internos, organização de camadas) é decisão sua, e faz parte da avaliação.

Convenções gerais:

- Recursos JSON sob `/api`, propriedades em `camelCase`.
- Erros de validação/regra respondem `400` (ou `409` quando indicado) com corpo
  `{ "error": "mensagem legível" }`. Recurso inexistente ou excluído → `404`.
- Datas em ISO 8601 UTC (`2026-06-30T14:00:00Z`).

---

## Parte 1 — Cadastro de contatos (obrigatório)

### `GET /api/contacts`

Lista paginada com busca.

| Query param | Default | Regra |
|---|---|---|
| `search` | — | opcional; filtra por nome **ou** e-mail, case-insensitive, correspondência parcial |
| `page` | `1` | 1-based; `400` se < 1 |
| `pageSize` | `20` | máximo `100`; `400` se fora de 1–100 |

```jsonc
// 200
{
  "items": [
    { "id": 42, "name": "Ana Lima", "email": "ana.lima@exemplo.com.br", "segment": "Plano Black" }
  ],
  "total": 320,     // total de contatos não excluídos que casam com a busca
  "page": 1,
  "pageSize": 20
}
```

A paginação acontece **no banco** (`LIMIT/OFFSET` ou equivalente), não em memória.

### `GET /api/contacts/{id}`

`200` com o contato, ou `404` se não existe/excluído.

```jsonc
{ "id": 42, "name": "Ana Lima", "email": "ana.lima@exemplo.com.br", "segment": "Plano Black" }
```

### `POST /api/contacts`

Corpo: `{ "name": "...", "email": "...", "segment": "..." | null }`.

- `201` + contato criado (com `id` gerado pelo banco) + header `Location`.
- `400` se `name`/`email` ausentes ou e-mail com formato inválido.
- `409` se já existe contato **não excluído** com o mesmo e-mail (case-insensitive).

### `PUT /api/contacts/{id}`

Mesmo corpo e validações do POST (o `409` de e-mail duplicado não pode acusar o
próprio contato). `200` + contato atualizado, ou `404`.

### `DELETE /api/contacts/{id}`

`204`. **Soft delete** — marca `deletedAt`; o contato some de todos os `GET`s e o
e-mail dele fica livre para novos cadastros. `404` se já não existe.

### `GET /api/contacts/{id}/responses`

Respostas válidas do contato (JOIN com pesquisas), mais recente primeiro. `404` se o
contato não existe/excluído.

```jsonc
// 200
[
  {
    "id": 815,
    "surveyId": 1,
    "surveyName": "NPS Pós-Treino",
    "surveyType": "NPS",
    "score": 9,
    "comment": "Atendimento excelente!",   // null quando não comentou
    "channel": "whatsapp",
    "respondedAt": "2026-05-12T18:22:10Z"
  }
]
```

---

## Parte 2 — Resumo de satisfação (obrigatório)

### `GET /api/analytics/summary`

Uma única consulta agregada sobre todas as respostas válidas do seed. As regras de
escala (NPS × CSAT) e arredondamento estão no [doc 01](01-regras-de-negocio.md).

```jsonc
// 200 — valores da tabela de conferência do doc 01
{
  "npsScore": 24,
  "npsResponses": 978,
  "promoters":  { "count": 477, "pct": 48.8 },
  "neutrals":   { "count": 256, "pct": 26.2 },
  "detractors": { "count": 245, "pct": 25.1 },
  "responsesCount": 1246,
  "csatAvg": 3.91          // null quando não há resposta CSAT no recorte
}
```

---

## Parte 3 — Bônus (opcionais)

> Escolha 1 ou 2. Cada um tem valores de conferência no doc 01 para você se validar.

### Filtros no resumo

`GET /api/analytics/summary?from=2026-06-01&to=2026-06-30&surveyId=1`

| Query param | Regra |
|---|---|
| `from` | data inicial inclusiva (`yyyy-MM-dd`); `400` se mal formatada |
| `to` | data final inclusiva; `400` se mal formatada |
| `surveyId` | restringe a uma pesquisa |

Todos opcionais e combináveis. Quando o recorte não tem nenhuma resposta NPS:
`npsScore` e as classes vêm zeradas — nunca `NaN`/erro 500. Se fizer este bônus,
exponha também `GET /api/surveys` (`[{ "id": 1, "name": "...", "type": "NPS" }]`)
para o dropdown do front.

### `GET /api/analytics/nps-by-month`

Só meses com pelo menos uma resposta NPS válida, em ordem cronológica (aceita os
mesmos filtros, se implementados):

```jsonc
[
  { "month": "2026-01", "responses": 104, "npsScore": 17 },
  { "month": "2026-02", "responses": 112, "npsScore": 26 }
]
```

### `GET /api/analytics/nps-by-segment`

Uma linha por segmento de contato (exige JOIN respostas × contatos), ordenado por
`npsScore` decrescente. Contatos sem segmento entram como `"Sem segmento"`.

```jsonc
[
  { "segment": "Plano Black", "responses": 415, "npsScore": 31 },
  { "segment": "Corporativo", "responses": 292, "npsScore": 19 },
  { "segment": "Plano Fit",   "responses": 271, "npsScore": 18 }
]
```

### `GET /api/responses/export`

CSV das respostas válidas, colunas `id,survey,contact,score,comment,channel,respondedAt`,
com escaping correto de vírgulas e aspas nos comentários.

---

## Checklist de sanidade do contrato

- [ ] `GET /api/analytics/summary` retorna `npsScore: 24` e `responsesCount: 1246`;
- [ ] `DELETE` de contato → some da lista, `GET /{id}` → `404`, e-mail liberado;
- [ ] `POST` com e-mail já usado → `409` com `{ "error": ... }`;
- [ ] `?search=` na lista não é sensível a maiúsculas;
- [ ] Nenhuma query monta SQL concatenando valor vindo do usuário.
