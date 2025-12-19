infrastructure/ (como faz de verdade)
DB async SQLAlchemy + models + migrations
Repositories concretos (Postgres)
Integrações:
    -integrations/github/ (GitHub API + validação de assinatura webhook)
    -integrations/osv/ (OSV API)
Fila:
    -RQ/Celery + workers
Segurança:
    -API keys, rate limit (Redis), hashing
Observabilidade:
    -logs estruturados, correlation-id, métricas
presentation/ (HTTP + validação + DI)
    -Rotas FastAPI, schemas (Pydantic), dependências
    -Middleware (correlation-id, auth, rate limit)
    -Endpoint de webhook