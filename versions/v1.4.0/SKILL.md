---
name: schematize-rust
metadata:
  version: 1.4.0
description: Padrões normativos de engenharia da casa com Rust como linguagem principal de backend e Go como auxiliar (arquitetura, segurança, testes, pentest, dados, observabilidade, deploy, archive). Use SEMPRE que for projetar, gerar, revisar ou refatorar qualquer backend, API, serviço, schema, migration, infra, CI/CD, teste ou deploy — mesmo que o usuário não cite "padrão" explicitamente. Aplique também ao decidir arquitetura, escolher stack, modelar eventos/banco, escrever testes/pentest, configurar observabilidade, ou produzir ADR/runbook/archive. Contém pisos inegociáveis de segurança (segredo nunca no cliente, sem SQL concatenado, auth server-side, archive obrigatório) e de código (≤300 linhas/arquivo, uma função por arquivo, tudo comentado, MAPA da aplicação) que vetam atalhos inseguros. Frontend delega ao schematize-web.
---

# Padrões de Engenharia da Casa — Rust principal

Conjunto normativo que rege como software de backend é projetado, construído, testado e operado aqui, **com Rust como linguagem principal e Go como auxiliar**. Mesma base normativa das demais skills da casa; o que muda é a ordem de preferência de linguagem e o ferramental.

**Versão:** skill `schematize-rust` v1.4.0. Changelog em `CHANGELOG.md`.

## Linguagem: Rust principal, Go auxiliar

- **Backend novo nasce em Rust** (Tokio/axum, sqlx, etc.). É a escolha padrão para serviços, APIs, workers e CLIs.
- **Go é a linguagem auxiliar**: use quando houver razão concreta — ecossistema/SDK maduro só em Go, time já dono do módulo, ou integração com base Go existente. A troca é decisão registrada em ADR, não preferência de gosto.
- **Node backend é proibido** (frontend Node é 100% permitido — ver `schematize-web`). **PHP é proibido** e migra sumariamente.
- Detalhe e critérios de migração em `references/stack-rust.md` e `references/arquitetura.md` (§3).

> Esta é a inversão da `schematize-go`: lá Go é principal e Rust a alternativa; aqui Rust é principal e Go o auxiliar. As duas podem coexistir na mesma máquina (comandos são namespaced por skill — ver abaixo).

## Comandos (Claude Code)

Digite `/rust-help` pra ver todos. Em resumo:

| Comando | O que faz |
|---|---|
| `/rust-help` | lista todos os comandos do schematize-rust |
| `/rust-cc` | context compact: gera context.md + checklist.md no archive e roda `/compact` |
| `/rust-handoff` | gera o handoff (context.md + checklist.md) **sem** compactar — pra fim de sessão |
| `/rust-qa` | fluxo de Q.A. plan-first (§22.9): planeja, gera MD, pede aprovação, roda |
| `/rust-review` | roda o gate da DoD no diff (arquivos >300, função sem doc, MAPA, índice, macaquices) |
| `/rust-index` | (re)gera o índice de microfunções a partir dos doc-comments |

Todos os comandos são prefixados com o slug da skill (`rust-`), então **convivem sem conflito** com `schematize-go` (`go-*`) e `schematize-web` (`web-*`) na mesma máquina. Ficam em `assets/commands/` e são instalados em `.claude/commands/`.

## Como usar esta skill

1. Identifique o domínio da tarefa e **leia o(s) reference(s) relevante(s)** antes de produzir código ou decisão. Não trabalhe de memória.
2. **Sempre** aplique os pisos inegociáveis abaixo, independente do reference carregado.
3. Ao terminar, valide contra a Definition of Done (`references/entrega.md`, §35) e **gere o archive** (§28, `references/operacao.md`).

Mapa de references — leia o que casa com a tarefa:

| Tarefa | Reference |
|---|---|
| **Stack Rust (principal), Go (auxiliar), toolchain, quando trocar de linguagem** | `references/stack-rust.md` |
| **Async/concorrência (Tokio): não bloquear o runtime, cancel-safety, locks, backpressure, shutdown** | `references/async-concorrencia.md` |
| **Limites de código (≤300 linhas), uma função/arquivo, comentários, MAPA** | `references/padroes-codigo.md` |
| Arquitetura, camadas, DDD, repositórios, anti-monólito, shared libs, CQRS | `references/arquitetura.md` |
| Eventos/mensageria, banco, cache, APIs, resiliência, jobs | `references/dados-eventos.md` |
| Segurança, auth, JWT, multi-tenancy, LGPD, segredos | `references/seguranca.md` |
| **Cadeia de suprimentos: lockfile, SBOM, scan que trava, imagem mínima/pinada/assinada, SLSA, segredo no build** | `references/cadeia-suprimentos.md` |
| Testes — test kit, saída machine-readable, categorias de teste (§22.1–22.3) | `references/testes.md` |
| Testes — padrão de script, seeds, CI, pentest, Q.A. plan-first, Makefile (§22.4–23) | `references/testes-execucao.md` |
| Observabilidade, healthchecks, performance, FinOps | `references/observabilidade.md` |
| Config, deploy/K8s, git/PR, ownership, runbooks/incidentes, ADR, **archive** (§20–28) | `references/operacao.md` |
| Templates, feature flags, IA assistida, DoD, evolução, índice de funcionalidades (§29+) | `references/entrega.md` |
| Filosofia, aplicação universal e a lista completa de anti-padrões vetados | `references/anti-padroes.md` |
| Gestão de contexto em sessões longas no Claude Code (handoff, hooks) | `references/contexto-claude-code.md` |

## Pisos inegociáveis (VETADO — sem ADR de exceção)

Nunca violados, nem "pra funcionar", nem "pra ir mais rápido". Lista completa em `references/anti-padroes.md`. Os que mais aparecem:

- **Segredo nunca no cliente.** Nada de API key, secret de JWT, senha de banco ou token em bundle do browser nem em `NEXT_PUBLIC_*`/`VITE_*`. Segredo só server-side. Detalhe em `references/seguranca.md`.
- **SQL sempre parametrizado.** Concatenar string em query é injeção esperando acontecer (em Rust, use `sqlx`/binds; nunca `format!` em SQL).
- **Auth e autorização server-side.** `tenant_id`/role/`user_id` vêm do token verificado, nunca do body/header do cliente.
- **JWT validado por inteiro** (assinatura, exp, aud, iss, alg em allowlist). Senha em argon2id/bcrypt cost ≥ 12. Token de sessão por CSPRNG.
- **Erro nunca engolido.** Sem `let _ = ...` pra calar resultado, sem `.unwrap()`/`.expect()` em caminho de produção pra silenciar o compilador; trate `Result`/`Option` de verdade (`?`, match, erro tipado).
- **Teste nunca silenciado** pra passar CI (`#[ignore]`, comentar assert, baixar threshold). Conserta o código, não o teste.
- **Sem monólito que mistura bounded contexts**, sem monólito distribuído, sem shared lib `commons` de domínio.
- **Archive SEMPRE gerado.** Toda entrega que produz código/decisão/mudança de estado gera o `.md` de archive. Templates em `assets/`.
- **Migration reversível** (com `down`, testada com rollback). Container não-root, read-only. Dependência nova com nome/licença/versão verificados.
- **Pisos de código (`references/padroes-codigo.md`):** arquivos **≤ 300 linhas** (acima → quebrar por coesão), **uma função/unidade lógica por arquivo** (função > 300 linhas é quebrada), **toda função com doc-comment** (`///`: motivo, comportamento esperado, entradas, saídas, efeitos), **`MAPA.md` da aplicação** atualizado no mesmo PR — em **`<projeto>_archive/index/`, nunca no root** — e **índice de microfunções** regenerado (`/rust-index`). **Todo MD gerado (MAPA/índice/plano/relatório/handoff) mora no archive**, root limpo (§28). Detalhe em `references/padroes-codigo.md` (§4) e `references/operacao.md` (§28, §39).
- **Backend novo prioriza Rust; Go é auxiliar; Node backend e PHP são proibidos.** Critérios em `references/stack-rust.md`.

> Regra de bolso: se a justificativa começa com "só pra funcionar", "depois eu arrumo" ou "é mais rápido assim" e o resultado mexe em segredo, auth, dado ou registro — é anti-padrão vetado. Pare e faça certo.

## Testes — o que conta como "verde de verdade"

Detalhe em `references/testes.md` (§22.1–22.3) e `references/testes-execucao.md` (§22.4–23). Essencial: smoke que assere shape (não só 200) e tem self-check que força um FAIL conhecido; unit agressivo com caminho de erro, casos hostis, property-based e mutation no domínio crítico; pentest que prova rejeição rota a rota; `simulated` que cruza rotas × personas × injections cobrindo 100% das rotas; e Q.A. plan-first (planeja, gera MD, pede aprovação antes de rodar).

## Andaime pronto (scripts e templates)

Não escreva do zero o que já está bundlado: `scripts/lib.sh`, `scripts/test-skeleton.sh`, `scripts/smoke-selfcheck.sh`, `scripts/simulated/run.py`, os hooks de contexto (`scripts/hooks/*.mjs`), os templates em `assets/` (ADR, TASK, CHAT_ARCHIVE, PR_TEMPLATE, RUNBOOK, **MAPA**), o índice (`assets/INDEX_*` + `scripts/build-index.mjs`) e `assets/CLAUDE.md` pra pinar os padrões na raiz do repo.

## Aplicação sempre-on

Copie `assets/CLAUDE.md` para a raiz do projeto pra garantir os padrões em toda interação do repo, não só quando a skill dispara.
