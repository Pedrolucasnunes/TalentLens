# TalentLens

![CI](https://github.com/Pedrolucasnunes/TalentLens/actions/workflows/ci.yml/badge.svg)
![Python](https://img.shields.io/badge/python-3.13-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-0.139-teal)
![License](https://img.shields.io/badge/license-MIT-green)

Análise de aderência entre currículo e vaga com IA. A partir dos dois textos, o TalentLens calcula um score de similaridade semântica e gera um parecer estruturado com pontos fortes, pontos de atenção e recomendação.

Pipeline em Python puro, sem framework de orquestração: embeddings, similaridade de cosseno, recuperação de análises anteriores (RAG com pgvector) e parecer via LLM.

## Como funciona

```
Currículo (texto) + Descrição da vaga (texto)
        ↓
Embeddings dos dois textos (OpenAI text-embedding-3-small)
        ↓
Score: similaridade de cosseno entre os dois embeddings (numpy, escala 0–100)
        ↓
Recupera as 3 análises anteriores de currículos mais parecidos (Postgres + pgvector)
        ↓
LLM (gpt-4o-mini) recebe vaga + currículo + score + análises anteriores → parecer em JSON
        ↓
{ score, parecer, pontos_fortes, pontos_fracos, recomendacao }
        ↓
Análise salva na memória vetorial para calibrar as próximas
```

A memória vetorial é opcional: sem banco configurado, a análise roda normalmente, apenas sem o contexto histórico ([detalhes](backend/README.md#memória-de-análises-rag)).

## Demonstração

![Demonstração do TalentLens](docs/screenshot-demo.png)

## Decisões técnicas

- **Sem framework de orquestração.** O fluxo é linear e tem poucas etapas, então a API da OpenAI é chamada diretamente. Cada etapa fica visível no código e fácil de testar com mocks. Um framework como LangGraph passaria a fazer sentido se o fluxo virasse um agente com várias ferramentas e estado.
- **pgvector em vez de um banco vetorial dedicado.** Os vetores e os dados da análise ficam na mesma tabela, consultados com SQL, sem mais um serviço para operar. O Supabase é usado só como Postgres gerenciado. Para volumes muito grandes, um banco vetorial dedicado tende a escalar melhor.
- **Score e parecer separados.** O score vem dos embeddings, é determinístico e barato. O LLM entra só no parecer qualitativo, recebendo o score como contexto.
- **Memória opcional por design.** Se o banco não estiver configurado ou falhar, a análise segue sem o contexto histórico. A memória melhora o resultado, mas nunca pode derrubar o endpoint principal.
- **Saída em JSON com temperatura baixa.** `response_format` em modo JSON e `temperature=0.3`, para respostas consistentes e fáceis de converter no schema da API.

## Limitações conhecidas

- O score é a similaridade de cosseno crua, não uma probabilidade calibrada. Na prática, textos reais raramente ficam perto de 0 ou de 100.
- A recuperação de análises usa só o embedding do currículo, sem considerar a vaga, e não aplica limiar mínimo de similaridade.
- As análises usadas como calibração são saídas do próprio LLM, sem revisão humana, o que pode reforçar erros.
- O modo JSON garante JSON válido, mas não o formato: a recomendação não é restrita aos três valores esperados.
- Analisa um currículo por vez, em texto. Não lê PDF nem ranqueia vários candidatos.
- Não tem autenticação nem limite de requisições, e armazena o texto completo do currículo. Para uso real, seriam necessárias políticas de retenção e anonimização (LGPD).

## Próximos passos

- Structured Outputs com schema, restringindo a recomendação aos valores válidos
- Considerar a vaga na recuperação de análises e aplicar um limiar de similaridade
- Usar como calibração apenas análises revisadas por uma pessoa
- Conjunto de avaliação com pares currículo × vaga de resultado conhecido, para medir a qualidade das recomendações
- Evoluir para um agente com tool calling, em que o modelo decide quando buscar análises anteriores ou consultar requisitos da vaga

## Arquitetura

| Camada | Stack | Papel |
|---|---|---|
| `backend/` | Python + FastAPI + OpenAI + Postgres (pgvector) | Análise de aderência currículo × vaga: embeddings, similaridade, RAG e parecer via LLM |
| `landing/` | HTML/CSS/JS | Landing page usada para validar interesse no produto (ver abaixo) |

O FastAPI serve tudo em um único servidor: landing page em `/`, interface de análise em `/app` e a API (`/analisar`, `/embeddings`, `/health`).

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

Interface em `http://localhost:8000/app` e docs interativas em `http://localhost:8000/docs`. Endpoints, exemplo de resposta e configuração da memória vetorial em [backend/README.md](backend/README.md).

## Testes

```bash
cd backend
pip install -r requirements-dev.txt
pytest -v
```

As chamadas à OpenAI e ao banco são simuladas: nenhum teste consome API nem exige chave ou banco real. O CI (GitHub Actions) roda a suíte a cada push e pull request.

## Sobre a landing page

A landing em `landing/` foi criada para validar o interesse de recrutadores antes de construir o produto. Ela descreve a visão do produto, não o que este MVP faz hoje: funcionalidades como upload em massa, integração com ATS e analytics não estão implementadas. O formulário de lista de espera apontava para um projeto Supabase que foi desativado e não funciona mais.

## Status

MVP funcional. A evolução planejada está em [Próximos passos](#próximos-passos).

## Licença

MIT
