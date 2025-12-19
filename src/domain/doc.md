domain/ (regra do jogo)
    - Entidades: Organization, Repository, DependencySnapshot, Finding, Policy
    - Value objects: RepoRef(owner,name), Ecosystem, Severity, Role
    - Serviços de domínio: avaliação de política, normalização de dependência, status do repo
    - Eventos de domínio: RepoConnected, ScanRequested, ScanCompleted