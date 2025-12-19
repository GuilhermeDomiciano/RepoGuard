# RepoGuard (open-source)
Monitoramento de dependências, SBOM simples e vulnerabilidades (OSV) para repositórios GitHub, com arquitetura Clean, multi-tenant, RBAC, audit log, jobs assíncronos e webhooks.

## Objetivo do projeto
Construir um backend em Python (provável FastAPI) com **infra real**, **dados reais**, foco forte em **arquitetura** e padrão de pastas inspirado no repositório `mcp-api-contact2sale`:

- Estrutura em `src/` separada por camadas:
  - `src/domain`
  - `src/application`
  - `src/infrastructure`
  - `src/presentation`
- Entry points na raiz (`main.py`, `worker.py`, etc.)
- Docker/Docker Compose para ambiente local e deploy fácil

## Escopo (o que esse projeto entrega)
### Core
1. **Conectar repositórios GitHub**
   - Registrar repo por URL ou `owner/name`
   - Armazenar metadados (provider, default branch, status)
2. **Ingestão de dependências**
   - Parse inicial de dependências (prioridade: `requirements.txt`)
   - Normalização: nome/ecossistema/versão/escopo
   - Snapshot por commit (quando disponível)
3. **Scanner de vulnerabilidades (dados reais)**
   - Consulta à API do **OSV.dev** para cada dependência
   - Persistência de vulnerabilidades e findings
4. **Políticas**
   - Regras por organização (ex.: bloquear severidade alta)
   - Avaliação automática do repositório: `OK / WARN / FAIL`

### Diferenciais de portfólio (nível pleno)
- **Multi-tenant** (organizações) e isolamento por `org_id`
- **RBAC** (roles: `owner`, `admin`, `viewer`)
- **Audit log** (quem fez o quê, quando)
- **Jobs assíncronos** (fila Redis + worker)
- **Webhook do GitHub** (push → re-scan)
- **Observabilidade** (correlation-id, métricas e/ou traces)
- **Rate limit + API keys** por organização

## Tecnologias (sugestão prática)
- API: **FastAPI**
- DB: **PostgreSQL**
- Queue: **Redis + RQ** (ou Celery)
- ORM: **SQLAlchemy 2.0** (ou SQLModel)
- Migrations: **Alembic**
- Observabilidade: **OpenTelemetry** + endpoint de métricas (Prometheus-friendly)
- Infra local: **Docker Compose**
- CI: **GitHub Actions**
- Deploy: Render/Fly.io/VPS (com Postgres gerenciado)

> Ajustes finos de stack ficam livres, mas a ideia é manter Python e “cara de produção”.

## Arquitetura (Clean / camadas)
### `domain/` (regras do negócio)
- Entidades:
  - `Organization`, `User`, `Membership`
  - `Repository`
  - `DependencySnapshot`, `Dependency`
  - `Vulnerability`, `Finding`
  - `Policy`
  - `AuditLog`
- Value Objects:
  - `RepoRef(owner, name)`, `Ecosystem`, `Severity`, `Role`
- Serviços de domínio:
  - Normalização de dependência
  - Avaliação de política
  - Cálculo de status do repo
- Eventos de domínio (opcional):
  - `RepoConnected`, `ScanRequested`, `ScanCompleted`

### `application/` (casos de uso)
- Orquestração do fluxo, sem FastAPI/SQLAlchemy
- Use cases:
  - `ConnectRepositoryUseCase`
  - `TriggerScanUseCase`
  - `IngestDependenciesUseCase`
  - `EvaluatePolicyUseCase`
  - `ListFindingsUseCase`
  - `HandleGithubWebhookUseCase`
- Ports (interfaces):
  - `RepositoryRepoPort`, `SnapshotRepoPort`, `FindingRepoPort`, `PolicyRepoPort`
  - `GithubClientPort`, `OsvClientPort`
  - `QueuePort`
  - `AuditPort`

### `infrastructure/` (implementações)
- DB: sessão, models, migrations
- Repositórios concretos (Postgres)
- Integrações:
  - `github/` (API + validação de assinatura de webhook)
  - `osv/` (client para consulta em lote quando possível)
- Queue:
  - enqueue + tasks
- Segurança:
  - API keys, rate limit, hashing/validação de tokens
- Observabilidade:
  - logs estruturados, correlation-id, métricas/tracing

### `presentation/` (HTTP)
- FastAPI routers v1, schemas, deps, middlewares
- Endpoint de webhook
- Healthcheck


## Modelo de dados (mínimo recomendado)
- `organizations`
- `users`
- `memberships` (user_id, org_id, role)
- `repositories` (org_id, provider, owner, name, default_branch, status)
- `dependency_snapshots` (repo_id, commit_sha, created_at)
- `dependencies` (snapshot_id, ecosystem, name, version, scope)
- `vulnerabilities` (osv_id, summary, severity, references, ranges…)
- `findings` (dependency_id, vulnerability_id, status, first_seen, fixed_version?)
- `policies` (org_id, rules jsonb)
- `audit_logs` (org_id, actor_id, action, metadata jsonb)

Opcional (forte para portfólio):
- `outbox_events` (org_id, type, payload, status)

## API (rotas sugeridas)
Base: `/api/v1`

### Auth / API Keys
- `POST /auth/api-keys` — cria API key por organização

### Organização / RBAC
- `GET /orgs/me`
- `POST /orgs/{org_id}/invite`
- `PATCH /orgs/{org_id}/members/{user_id}` — altera role

### Repositórios
- `POST /orgs/{org_id}/repos` — conecta repo (owner/name ou URL)
- `GET /orgs/{org_id}/repos`
- `GET /orgs/{org_id}/repos/{repo_id}`

### Scans / Findings
- `POST /repos/{repo_id}/scan` — força scan
- `GET /repos/{repo_id}/findings?severity=high&status=open`
- `GET /repos/{repo_id}/snapshots/{snapshot_id}`

### Policies
- `PUT /orgs/{org_id}/policies`
- `GET /orgs/{org_id}/policies/evaluation?repo_id=...`

### Webhook
- `POST /webhooks/github` — recebe push/PR (mínimo: push) e enfileira re-scan

### Health
- `GET /health`

## Jobs assíncronos (worker)
- Job principal: `scan_repo(repo_id)`
  1. Busca último estado do repo (ou usa payload do webhook)
  2. Gera snapshot
  3. Extrai dependências (requirements.txt)
  4. Consulta OSV
  5. Persiste findings e recalcula status por policy

## Segurança
- Isolamento por tenant (`org_id` em tudo)
- RBAC no nível de rota/use case
- API keys por organização (para uso como serviço)
- Rate limit por key via Redis

## Observabilidade
- Correlation-id por request
- Logs estruturados
- Métricas mínimas:
  - duração de scan
  - scans por status
  - falhas por integração (GitHub/OSV)
  - tamanho da fila

## Infra local (Docker Compose)
Serviços:
- `api`
- `postgres`
- `redis`
- `worker`

## CI/CD (GitHub Actions)
- Lint/format
- Testes unit + integration
- Build de imagem Docker (opcional)
- Deploy (opcional)

## Plano de execução (2–3 semanas, poucas horas)
### Semana 1
- Base do projeto + Docker Compose
- Migrations + modelos principais
- Multi-tenant + RBAC + audit log
- Endpoint de conectar repositório
- Ingestão inicial de `requirements.txt`

### Semana 2
- Worker + fila
- Scan OSV + findings + endpoints de listagem
- Webhook push → enqueue re-scan
- Policies simples + status do repo

### Semana 3
- Observabilidade (mínimo viável)
- Rate limit + API keys
- Testes + CI
- Deploy simples
- README “bonito” (diagramas + tradeoffs + exemplos)

## Roadmap (se sobrar tempo)
- SBOM export (CycloneDX)
- Suporte a `poetry.lock` e `package-lock.json`
- Notificações (Discord webhook/email) quando repo ficar `FAIL`
- Outbox pattern para eventos internos
- Incremental scan por commit (otimização)

## Licença
Definir (ex.: MIT/Apache-2.0) e manter o projeto como open source.
