# Changelog — schematize-rust

Formato: [Keep a Changelog]; versionamento: SemVer. Inversão da `schematize-go`:
aqui **Rust é a linguagem principal de backend e Go a auxiliar**. Frontend delega
ao `schematize-web`.

## [1.3.0] — 2026-07-03

### Alterado
- **Índice/MAPA exaustivo e como grafo** (§4 / §39 / `/rust-index` / `MAPA.md` / `CLAUDE.md`): o índice passa a exigir **uma entrada por função** de cada serviço/app (`nº entradas == nº funções`). O `/rust-index` **conta as declarações** e **reprova** se o índice tiver menos entradas, listando as ausentes pelo nome — chega de mapa magro (o caso "90 linhas pra 100+"). Removida a brecha do "relevante". O MAPA vira **grafo** (serviços + chamadas, Mermaid + adjacência), não lista.

## [1.2.0] — 2026-07-03

### Adicionado
- **Contenção no workspace** (§2 / anti-padrões §37 / `CLAUDE.md`): aplicação/repo novo nasce **dentro da pasta do projeto atual** (`./<projeto>_<contexto>/`). Veto a começar largando arquivos no root e depois **subir de diretório** (`cd ..`, `../`) pra criar repos irmãos fora, ou espalhar arquivos em `~`/`Documents`/`Downloads`/`/tmp`/Área de Trabalho. O agente **não sai da pasta do projeto** (ler ou escrever) sem o usuário pedir.

## [1.1.1] — 2026-06-27

### Alterado
- `/rust-claude` passa a **mesclar** o `CLAUDE.md` em repo multi-linguagem (Node/Go/Rust/Web) — não sobrescreve blocos de outras skills; nota de convivência no `CLAUDE.md`.

## [1.1.0] — 2026-06-27

### Adicionado
- **Convenção de nome de repositório** `<projeto>_<contexto>[_<lang>]` e o repo **`<projeto>_ops`** (control plane de desenvolvimento) — arquitetura §2.
- **Independência de runtime** (cada serviço sobe e funciona sozinho; nunca crash em cascata) + **resiliência store-and-forward** (persiste/loga/alerta/retoma na falha de notificação a outro serviço) — §2/§18; anti-padrões 32/33.
- **Observabilidade LGTM+** integrada em toda ferramenta/serviço (OpenTelemetry → Grafana Alloy → Loki/Tempo/Prometheus/Mimir → Grafana, + Helm chart) — §16.
- **Doc-comment com fluxo do dado** (de onde vem → o que faz → pra onde vai, inclusive cross-aplicação) e "todas as funcionalidades mapeadas" — §3/§39.
- Comandos: **`/rust-load`** (carrega à força o corpo normativo) e **`/rust-claude`** (cria/atualiza o `CLAUDE.md` da raiz).
- `CLAUDE.md` sempre-on atualizado (pisos 10 e 11).

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
