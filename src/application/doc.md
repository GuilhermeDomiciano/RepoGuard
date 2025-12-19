application/ (casos de uso)
Use cases (orquestram tudo, não conhecem FastAPI nem SQLAlchemy):
    -ConnectRepositoryUseCase
    -TriggerScanUseCase
    -IngestDependenciesUseCase
    -EvaluatePolicyUseCase
    -ListFindingsUseCase
    -HandleGithubWebhookUseCase
Ports (interfaces):
    -RepositoryRepoPort, SnapshotRepoPort, FindingRepoPort, PolicyRepoPort
    -GithubClientPort, OsvClientPort
    -QueuePort (enqueue job)
    -AuditPort