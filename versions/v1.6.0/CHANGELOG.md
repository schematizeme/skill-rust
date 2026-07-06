# Changelog — schematize-rust

Formato: [Keep a Changelog]; versionamento: SemVer. Inversão da `schematize-go`:
aqui **Rust é a linguagem principal de backend e Go a auxiliar**. Frontend delega
ao `schematize-web`.

## [1.6.0] — 2026-07-06
Deploy destrutivo por seed + isolamento por usuário (ops).

### Adicionado
- references/ops.md §2: layout /<app>/ + repos dentro; /<app>/.env seeder global; redeploy destrutivo na app (preserva dados; ops reset gated dev/hml).
- references/ops.md §3: isolamento por usuário (user Linux + systemd hardened por serviço).
- Piso de seed/isolamento no CLAUDE.md; anti-padrões; /rust-ops audita layout/seed/isolamento.

## [1.5.0] — 2026-07-05
Control plane <projeto>_ops: fluxo de ambientes, ops interface única, instalação paralela, independência invariante.

### Adicionado
- references/ops.md: fluxo dev→local→github→hml→prd (nada direto no servidor), ops interface única (100%, autônomo), instalação paralela=nproc, independência invariante (falha no paralelo = serviços não independentes → prioridade máxima).
- Comando /rust-ops; pisos de ambientes/ops no CLAUDE.md; anti-padrões (editar no servidor, pular pra hml/prd, operar fora do ops, instalar serial, serializar pra mascarar); operacao.md §21 estendido; /rust-load carrega ops.md.

## [1.4.0] — 2026-07-05
Todo MD gerado no archive, root limpo.

### Corrigido
- MAPA/índice saíam no root → agora `<projeto>_archive/index/` (padroes-codigo §4, MAPA.md, /rust-index, build-index.mjs, CLAUDE.md, SKILL.md).

### Adicionado
- §28.0 (operacao.md): layout canônico do archive — todo MD gerado em `<projeto>_archive/<área>/`, NUNCA no root.

## [1.0.0] — 2026-06-20
Primeira release do **schematize-rust** — padrões normativos de engenharia da casa
com Rust como escolha padrão de backend e Go como auxiliar.

### Adicionado
- Conhecimento normativo fatiado em `references/` (stack Rust/Go, async/concorrência
  com Tokio, padrões de código, arquitetura, dados/eventos, segurança, cadeia de
  suprimentos, testes/pentest, observabilidade, operação/entrega, anti-padrões,
  contexto Claude Code).
- Comandos: `/rust-help`, `/rust-cc`, `/rust-handoff`, `/rust-qa`, `/rust-review`,
  `/rust-index` — prefixados por `rust-`, **sem conflito** com `schematize-go`
  (`go-*`) e `schematize-web` (`web-*`) na mesma máquina.
- Scripts: `lib.sh`, `test-skeleton.sh`, `smoke-selfcheck.sh`, `simulated/run.py`,
  `build-index.mjs`, `check-diff.sh`, `archive-secret-scan.sh`, hooks de contexto.
- Assets: `CLAUDE.md`, templates (ADR/TASK/CHAT/PR/RUNBOOK/MAPA/INDEX_*),
  `settings.claude.example.json`, CI (`ci/`), lint (`lint/`), pre-commit (`hooks/`).

### Pisos inegociáveis cobertos
- Backend novo prioriza **Rust** (Tokio/axum, sqlx); **Go é auxiliar** (decisão em ADR);
  Node backend e PHP proibidos; frontend Node permitido (ver `schematize-web`).
- Segredo nunca no cliente; SQL parametrizado (`sqlx`/binds, nunca `format!` em SQL);
  auth/authz server-side; JWT validado por inteiro.
- Erro nunca engolido (`let _`/`.unwrap()`/`.expect()` em produção vetados); trate
  `Result`/`Option` de verdade.
- Async correto (Tokio): não bloquear o runtime, cancel-safety em `select!`, locks
  nunca cruzando `.await`, backpressure, graceful shutdown.
- Pisos de código: arquivos ≤300 linhas, uma função/arquivo, doc-comment (`///`)
  obrigatório, `MAPA.md` e índice de microfunções atualizados.
- Cadeia de suprimentos: lockfile, SBOM, scan que trava (cargo audit/deny), imagem
  mínima/pinada por digest/não-root/assinada (cosign), SLSA.
- Archive obrigatório (§28); migration reversível; Q.A. plan-first (§22.9);
  handoff de contexto (§34.1).
