# Changelog — schematize-rust

Formato: [Keep a Changelog]; versionamento: SemVer. Inversão da `schematize-go`:
aqui **Rust é a linguagem principal de backend e Go a auxiliar**. Frontend delega
ao `schematize-web`.

## [1.9.1] — 2026-08-18
Q.A. repointado para a skill dedicada **schematize-qa**.

### Mudado
- **`/rust-qa` virou wrapper fino** da **schematize-qa** (`/qa-plan` → `/qa-run`) no recorte Rust (`cargo test`/`cargo nextest`). Referências ao antigo **§22.9** removidas de `SKILL.md`, `references/testes*.md`, `assets/CLAUDE.md` e `/rust-help`.

## [1.9.0] — 2026-08-15
Correção de desenho no IAM backend — **senha + Email OTP já é 2FA baseline** (o middleware/PEP não barra mais o login) + **risk engine adaptativo robusto**.

### Mudado (correção de piso)
- **`references/iam.md` §3/§4/§7 + roadmap + checklist:** senha + Email OTP conta como **2FA baseline** desde o cadastro; o **middleware/PEP libera o acesso baseline** e exige AAL alto **só por rota sensível** (step-up just-in-time, `403 step_up_required`), **nunca barra o login** por falta de fator forte — fim do **círculo infinito**. Fator forte é **nudge + just-in-time**. Migração ativa o Email OTP always-on como 2º fator baseline (sem muro).

### Adicionado
- **Autenticação adaptativa por risco (robusta)** (`references/iam.md` §9): log de sessões/tentativas + **score** (IP/ASN, device novo, geovelocidade, velocity, honeypot); **escalonamento 2FA→3FA** sob risco (senha → email → app/chave); **negação deceptiva/tarpit** (falso negativo sob suspeita — resposta idêntica ao erro real em **tempo constante**, estado "próxima passa" curto/escopado a conta+IP+device, soma-se ao 3FA, nunca trava o legítimo); **honeypot**; notifica login suspeito + "não fui eu".

## [1.8.0] — 2026-07-11
IAM por desenho (angle Rust) — identidade + autorização robustas, auth como microserviço Rust separado.

### Adicionado
- **`references/iam.md`** — piso de IAM da casa especializado para **backend Rust**: **auth é app SEPARADA** — microserviço Rust `<projeto>_auth_rs` (axum/actix sobre Tokio) + front próprio em `auth.<domain>`, isolado (user/systemd próprios; nunca monolith; apps delegam por OIDC/OAuth2.1 + PKCE; chave de assinatura só no auth, JWKS público via `jsonwebtoken`/`josekit`). **ID interno imutável (ULID/UUIDv7) — email/telefone nunca é ID** (schema `users` + `identifiers` 1..N; N emails por usuário incentivado; SSO com recuperação local forçada; nudge de email secundário com detecção de provedor + tooltip). **≥2 fatores sempre** (AAL/NIST 800-63B): **passkey/WebAuthn no núcleo** (`webauthn-rs`), TOTP/push (`totp-rs`), **email OTP Resend always-on inclusive HML**, **Twilio** p/ telefone (providers plugáveis por trait); senha por padrão (**argon2id** via crate `argon2` em `spawn_blocking` + HIBP) mas opcional no seletor; invariante de troca "fator Y≠X no maior AAL"; **recuperação ≥ força do login**. **Multi-tenant RBAC/ABAC granular via ReBAC** (OpenFGA/SpiceDB): deny-default, **PEP como `tower` layer/extractor**, enforcement server-side, token fino, decisão auditada. **Multi-dispositivo** + view de remover; **sessão 7d/90d** (fim do "15 min e é chutado"); **logout irreversível** (revoga refresh+família, `jti` na denylist Redis/DB, apaga sessão server-side, não só cookie). **Migração de auth legado = prioridade 0** (strangler-fig, re-hash preguiçoso → argon2id). Rotina agressiva de testes cross-tenant/priv-esc (schematize-pentest).
- **Comando `/rust-iam`** (plan-first): força/audita/scaffolda o IAM num projeto (bootstrap como microserviço Rust) ou porta um auth legado (migrate).
- **Piso 16** no `CLAUDE.md`; bullet nos pisos + linha na tabela de references + `/rust-iam` na tabela de comandos do `SKILL.md` (incl. `description:` do frontmatter); anti-padrões **43–46** (auth monolith; email como ID / 1 fator; authz hand-rolled/no cliente; logout que só apaga cookie); `/rust-load` carrega `iam.md`; `/rust-help` lista `/rust-iam`.

## [1.7.0] — 2026-07-11
Limite de arquivo em camadas — teto de 750 (≤500 úteis + ~250 comentário) + flag em >300 úteis.

### Alterado
- **`references/padroes-codigo.md` §1/§2:** o limite rígido de **300 linhas/arquivo** vira regra **em camadas**. **Teto DURO: 750 linhas** (das quais **~250 reservadas a comentário/doc** e **até ~500 de código útil**) — acima bloqueia. **FLAG (não bloqueia, mas SEMPRE sinaliza) em > 300 linhas de código útil:** indício de que a função está **muito extensa** / **precisa de mais abstração** — registra como dívida e **revê quando as prioridades forem resolvidas**. **Observabilidade tem folga natural (~400 úteis).** Função com >300 úteis dispara o mesmo flag; "uma função por arquivo" mantida.
- **`scripts/check-diff.sh`:** o gate de tamanho passa a contar **código útil** (exclui comentário/branco): `total > 750` **bloqueia**, `útil > 500` **bloqueia**, `útil > 300` (ou `> 400` em arquivo de observabilidade) **flagueia** (`warn`, não trava).
- Propagado no piso do `CLAUDE.md`, `SKILL.md` (incl. `description:`), `references/entrega.md` (DoD), `references/arquitetura.md` (§6), `references/stack-rust.md` e comandos `/rust-load` `/rust-help` `/rust-review`.

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
