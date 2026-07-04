# 03 · Critérios de avaliação

Transparência total: é exatamente assim que seu desafio será pontuado. A nota decide a
passagem para a próxima fase (conversa técnica sobre o seu código).

---

## Pesos

| Eixo | Peso | O que olhamos |
|---|---|---|
| **1. Corretude das regras de negócio** | 25 | Valores do resumo batem com a tabela de conferência; `deletedAt` filtrado em toda query; classificação e arredondamento do NPS corretos; escalas NPS × CSAT nunca misturadas; validações do cadastro (e-mail único → 409, liberado após exclusão). |
| **2. Banco de dados & SQL** | 20 | Schema bem modelado (tipos, FKs, índice onde importa); importação do seed reproduzível; agregações feitas em SQL (`GROUP BY`/`CASE`/`JOIN`), não em memória; **parâmetros nomeados sempre** — zero SQL por concatenação; paginação e busca no banco. |
| **3. Backend — API REST** | 15 | Contrato implementado fielmente (shapes, status codes, erros com corpo JSON); separação de responsabilidades (endpoint fino, acesso a dados isolado); C# idiomático. |
| **4. Frontend** | 20 | Componentização com fronteiras sensatas; TypeScript de verdade (sem `any` espalhado); data fetching organizado (camada de API, não `fetch` solto em todo componente); estado de servidor gerenciado com clareza; busca/paginação/CRUD refletidos na tela sem recarregar. |
| **5. Experiência de uso** | 10 | Telas agradáveis e legíveis; loading/erro/vazio tratados; busca e paginação com feedback; confirmação antes de excluir; ninguém precisa abrir o console para entender o que aconteceu. |
| **6. Comunicação** | 10 | README com setup que funciona de primeira (banco incluído) e decisões justificadas; histórico de commits pequeno e descritivo; declaração de uso de IA. |

**Total: 100 pontos.**

## Como interpretamos a nota

A régua depende do nível da vaga em que você está concorrendo:

| Nota | Vaga júnior | Vaga pleno |
|---|---|---|
| **75+** | avança | avança |
| **60–74** | avança se os eixos 1 e 2 estiverem fortes | análise caso a caso |
| **< 60** | não avança neste momento | não avança neste momento |

Para **pleno**, esperamos também sinais de profundidade: pelo menos um bônus bem feito
(testes são o mais valorizado), decisões de modelagem justificadas no README e nenhum
dos itens da lista "o que derruba a nota" abaixo.

Bônus (filtros no resumo, NPS por mês com gráfico, NPS por segmento, testes, export
CSV, Docker, CI) somam até **10 pontos extras** — mas não compensam eixo obrigatório
zerado.

## O que derruba a nota rápido

- Valores de conferência não batendo (em especial por `deletedAt` ignorado);
- **SQL montado com concatenação/interpolação de entrada do usuário** — reprova o
  eixo 2 inteiro, independente do resto;
- Agregações feitas em memória (carregar a tabela e contar com LINQ/JS);
- ORM completo (EF Core) no lugar do Dapper;
- Contrato diferente do especificado (shapes, status codes, nomes de campos);
- `any` como tipo padrão no front; lógica de negócio dentro do endpoint;
- Projeto que não sobe seguindo o próprio README;
- Commit único gigante ("versão final");
- Crash na tela quando a API retorna erro.

## O que impressiona

- Testes cobrindo as consultas de analytics contra os valores de conferência;
- Índices com comentário explicando o porquê (ex.: busca por e-mail, filtro por data);
- Paginação e busca fluidas (debounce, estado preservado);
- Tratamento decente de estados intermediários (skeleton, retry, empty state com CTA);
- README com seção "o que eu faria diferente com mais tempo" honesta.

---

Lembrete: na fase seguinte vamos conversar sobre **o seu código e o seu SQL** —
escolhas, alternativas que você descartou e o que mudaria. Escreva o desafio pensando
nessa conversa.
