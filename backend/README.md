# TalentLens: Backend

API em Python (FastAPI) que analisa a aderência de um currículo a uma vaga:

```
currículo + vaga → embeddings (text-embedding-3-small) → similaridade de cosseno (numpy)
                 → recupera análises anteriores similares (Postgres + pgvector, se configurado)
                 → LLM (gpt-4o-mini) gera parecer estruturado, calibrado pelo histórico
                 → score + parecer + pontos fortes/fracos + recomendação
                 → análise salva na memória vetorial para calibrar as próximas
```

## Rodar localmente

```bash
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env             # Windows: copy .env.example .env
# preencher OPENAI_API_KEY no .env
uvicorn main:app --reload
```

O servidor serve tudo em um lugar só:

| Rota | O que é |
|---|---|
| `/` | Landing page |
| `/app` | Interface de análise (colar vaga + currículo → análise) |
| `/docs` | Docs interativas da API (Swagger) |

## Endpoints da API

| Endpoint | Entrada | Saída |
|---|---|---|
| `POST /analisar` | `{curriculo, vaga}` | `{score, parecer, pontos_fortes, pontos_fracos, recomendacao}` |
| `POST /embeddings` | `{texto}` | `{modelo, dimensoes, embedding}` |
| `GET /health` | (sem entrada) | `{status: "ok"}` |

### Exemplo

```bash
curl -X POST http://localhost:8000/analisar \
  -H "Content-Type: application/json" \
  -d '{"curriculo": "Desenvolvedor Python com 3 anos de experiência em FastAPI e PostgreSQL...", "vaga": "Vaga para desenvolvedor backend Python pleno com FastAPI..."}'
```

Resposta:

```json
{
  "score": 78.4,
  "parecer": "O candidato tem boa aderência técnica à vaga, com experiência sólida em FastAPI e PostgreSQL, o stack principal pedido. A senioridade declarada é compatível com a posição de nível pleno.",
  "pontos_fortes": [
    "Experiência direta com o stack pedido (FastAPI, PostgreSQL)",
    "Histórico de projetos em produção"
  ],
  "pontos_fracos": [
    "Não menciona experiência com filas ou processamento assíncrono"
  ],
  "recomendacao": "Avançar para entrevista"
}
```

O `score` é a similaridade de cosseno entre os embeddings do currículo e da vaga, em escala de 0 a 100. O parecer é gerado pelo LLM recebendo os dois textos e o score como contexto.

## Memória de análises (RAG)

Não é um endpoint novo: é o `POST /analisar` enriquecido. Quando a memória vetorial está configurada, cada análise:

1. **Recupera** do Postgres (pgvector) as 3 análises anteriores cujos currículos são mais similares ao atual (`memoria.buscar_contexto_similar`)
2. Inclui os pareceres delas no prompt do LLM como **calibração**: vaga, score e recomendação de cada uma
3. **Salva** a análise concluída (embedding + parecer) para servir de contexto às próximas (`memoria.salvar_analise`)

**Degradação graciosa:** sem `SUPABASE_URL`/`SUPABASE_SERVICE_KEY` no `.env`, ou com o banco fora do ar, o `/analisar` funciona normalmente, só que sem o contexto histórico. A ausência da memória nunca quebra a análise.

Para ativar:

1. Crie um projeto no [Supabase](https://supabase.com) (usado só como Postgres gerenciado com pgvector)
2. Rode [supabase/migrations/001_analises_vetoriais.sql](supabase/migrations/001_analises_vetoriais.sql) no SQL Editor do projeto
3. Preencha `SUPABASE_URL` e `SUPABASE_SERVICE_KEY` no `backend/.env`

## Testes

```bash
pip install -r requirements-dev.txt
pytest -v
```

As chamadas à OpenAI e ao banco são simuladas nos testes: nada de custo de API, chave ou banco real necessários. O CI (GitHub Actions) roda a suíte a cada push e pull request.

## Configuração

| Variável | Obrigatória | Descrição |
|---|---|---|
| `OPENAI_API_KEY` | sim | Chave da OpenAI (embeddings + parecer) |
| `SUPABASE_URL` | não | URL do projeto Supabase (memória vetorial de análises). Sem ela, a análise roda sem contexto histórico. |
| `SUPABASE_SERVICE_KEY` | não | Service key do projeto Supabase (par da variável acima) |
| `CORS_ORIGENS` | não | Origens permitidas para CORS, separadas por vírgula. Padrão: só a landing publicada. Same-origin (landing e `/app` servidos pelo próprio FastAPI) não precisa de CORS. |
