# Testes — Execução, Pentest e Q.A.

> Parte da skill **schematize-rust**. Continuação de `testes.md` (numeração **preservada**, §22.4+): padrão de script de teste, seeds/personas, integração com CI, pentest (§22.7–22.8), fluxo de Q.A. plan-first (§22.9) e o Makefile padrão (§23).

---

### 22.4 Padrão de script (`tests/<mode>/<name>.sh`)

Skeleton obrigatório:

```bash
#!/usr/bin/env bash
# <Categoria> · <Nome curto>
# Descrição clara do que cobre e o esperado.
# Esperado: status X em caso Y. Falha = significado Z.

set -uo pipefail
_DIR="$(cd -P "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
# shellcheck source=../lib.sh
source "$_DIR/lib.sh"

test_section "<Categoria> · <Nome>"

API="$(api_base)"

# Caso 1
assert_http_in "<descrição do caso>" "200|401" GET "$API/v1/<rota>"

# Caso 2 — body shape
http_call GET "$API/v1/<rota>"
if [[ "$HTTP_CODE" == "200" ]]; then
  if echo "$HTTP_BODY" | jq -e '.data | length > 0' >/dev/null 2>&1; then
    test_pass "<rota> retorna dados"
  else
    test_fail "<rota> sem 'data'" "$HTTP_BODY"
  fi
fi

test_summary "<categoria>/<nome>"
exit $TEST_EXIT_CODE
```

**MUST**
- Banner do `test_section` na primeira linha do output.
- Cada caso assertado vira uma linha `✓` ou `✗` ou `○ skip`.
- `test_summary` no fim agrega contadores pra o runner.
- Falhas incluem `HTTP_BODY` truncado nos primeiros 2000 chars (sem PII).

---

### 22.5 Seeds e personas de teste

**MUST**
- Personas declaradas em `tests/seeds/test-*.sql` ou `tests/<mode>/*.json` — versionadas, reproduzíveis.
- Senhas test-only **explicitamente flagadas** (ex.: prefixo `SimTest!`) e **rejeitadas em produção** por validador de senha fraca.
- Setup idempotente: `INSERT ... ON CONFLICT DO UPDATE` — re-rodar não duplica.
- Cleanup automático no fim de cada run? **Não** — deixa lixo pra inspeção. Limpe via comando explícito (`<project> test clean-seeds`).

**Personas mínimas pra cobrir RBAC + multi-tenancy**:

1. `superadmin` — platform role, vê tudo.
2. `tenant_admin_A` — escopo do tenant A.
3. `tenant_admin_B` — escopo do tenant B (pra testar isolamento).
4. `normal_user` — sem role, só dados próprios.
5. (Opcional) `viewer`, `editor`, `support` — combinações de roles dentro de A.

---

### 22.6 Integração com CI

**MUST**
- PR check: `<project> test smoke + security + authz + hardening` (rápido — ~3min total).
- Pré-deploy: `<project> test all` (full minus chaos+unit — ~6min).
- Nightly: `<project> test full` (inclui chaos + unit).
- Bloqueio de merge: fail no `summary.json` (`.totals.fail > 0`) trava merge.

**Dashboards**
- `summary.json` é parseado por job de métricas pra empurrar `tests_pass_total`, `tests_fail_total`, `tests_duration_seconds_bucket` pra Prometheus.
- Falha em smoke pós-deploy dispara rollback automático.

---

### 22.7 Pentest leve via ferramentas externas (complementar)

**SHOULD** — em CI noturno, não a cada PR:
- **OWASP ZAP baseline** (`zap-baseline.py -t https://api.example.com`) — passive scan.
- **Nuclei** (`nuclei -u https://api.example.com -t cves,exposures,misconfiguration`).
- **trivy fs** ou **grype** sobre o repo + imagens Docker pra CVE em deps + base images.
- **Semgrep** ruleset `p/owasp-top-ten` pra SAST.

Resultados em PR comment ou dashboard, **não bloqueiam merge** por default (alto ruído) — mas qualquer **Critical** sem ADR de aceite trava.

---

### 22.8 Diretrizes de Pentest (princípios, não só scripts)

Os scripts da §22.3 são a implementação; estes são os princípios que regem **como** se faz pentest interno e **o que conta como passar**.

**Postura**
- **Assume-breach / hostil por padrão.** Todo input vindo de fora (body, query, header, cookie, path, arquivo, webhook) é tratado como hostil até prova de validação. Não existe "campo interno confiável" exposto numa rota.
- **Cobrir a rota toda, não a amostra.** Pentest roda contra o inventário completo de rotas (mesmo catalog do `simulated`). Rota nova sem caso de pentest correspondente bloqueia (liga com §35).
- **Caixa-cinza.** O time tem o OpenAPI, o schema e as roles — testa sabendo o que **deveria** acontecer, não no escuro. Pentest sem mapa vira teatro de "tentei o `' OR 1=1` e deu 403, tá seguro".

**Critério de resultado — o que é PASS / FAIL**
- **Nunca `500` por input malicioso.** Erro do servidor (5xx) diante de payload hostil é FALHA — significa que o input chegou fundo demais sem validação. O esperado é `400`/`422` (input inválido) ou `401`/`403`/`404` (sem acesso).
- **Nunca eco sem escape.** XSS/template injection refletido no body/header sem encode = FALHA, mesmo com status 200.
- **Nunca coerção silenciosa.** `varchar` aceito como `int`, string vazia virando `0`, `"true"` virando bool = FALHA (liga com `type-confusion.sh`). O tipo do schema é contrato.
- **Nunca vazamento entre tenants/usuários.** Qualquer `200` com dado de outro escopo = FALHA crítica, para o deploy na hora (§15).
- **Nunca diferença observável que ajude o atacante.** Mensagem de erro distinta pra "user existe" vs "senha errada", timing > 30% de variância, stack trace no response = FALHA.
- **Idempotência sob ataque.** Replay, duplicação, race no mesmo recurso não corrompe estado.

**Severidade e gate**
- Classificar todo achado: `Critical` / `High` / `Medium` / `Low` / `Info`.
- **`Critical` e `High` bloqueiam merge/deploy.** Sem exceção silenciosa — ou conserta, ou ADR de aceite de risco **com prazo de remediação e dono** (e mesmo assim `Critical` de auth/tenant/injeção **não aceita ADR** — é piso da §37).
- `Medium`/`Low` viram issue rastreável com prazo; acumular não é opção.

**Higiene**
- **Pentest roda só contra `dev`/`staging`** (ou alvo autorizado). Scripts destrutivos (`service-kill`, `db-disconnect`) gated por env var explícita (§22.3 chaos). Nunca contra produção sem autorização formal e janela.
- **Evidência sempre.** Todo achado tem request/response reproduzível no `raw.jsonl` — "achei uma falha" sem repro não conta.
- **Falso-positivo é bug do teste.** Pentest que grita sem motivo treina o time a ignorar — calibra ou remove. Mas **na dúvida, FAIL** (vai pra `REVIEW`, olho humano decide), nunca silencia por padrão.

**Escopo mínimo coberto** (mapeia OWASP Top 10 + API Top 10): injeção (SQL/NoSQL/command/template), XSS, validação de tipo e sanitização (§22.3), authz/IDOR/BOLA, broken auth (JWT, sessão, refresh), SSRF, mass assignment, exposição excessiva de dados, misconfig (headers, CORS, paths expostos), rate limiting, e abuso de recurso (payload gigante, ReDoS, billion-laughs).

> Pentest não "tenta hackear pra ver se acha algo". Ele **prova, rota por rota, campo por campo, que o sistema rejeita o que deve rejeitar.** Achado é evidência; ausência de achado só vale se a cobertura for total.

---

### 22.9 Fluxo de execução de Q.A. (plan-first, aprovação obrigatória)

A malha de Q.A. inclui passos potencialmente destrutivos (chaos `service-kill`/`db-disconnect`, pentest, mutações). Por isso **nenhuma submissão de Q.A. roda às cegas**: toda submissão de um formulário/pedido de Q.A. passa por planejamento, aprovação humana e execução controlada.

**MUST — antes de executar qualquer coisa**

1. **Planejar tudo primeiro.** Ao receber a submissão, o agente **não executa nada ainda**. Levanta o escopo completo: quais modos vão rodar (smoke/integration/security/pentest/authz/hardening/chaos/simulated/unit), ambiente alvo, rotas e personas afetadas, ordem de execução, dependências entre passos, o que é **destrutivo/gated**, e os riscos.
2. **Gerar um MD de passo a passo detalhado** em `<project>_archive/qa/<YYYY-MM-DD-HH-MM-SS>-<contexto>.md` (liga com §28 — é archive obrigatório). Cada passo declara: objetivo, comando exato, ambiente, resultado esperado, critério de pass/fail, e flag de **destrutivo** quando aplicável. O plano referencia o `summary.json` (§22.2) que será produzido.
3. **Pedir aprovação explícita do usuário.** Sem aprovação registrada, **nada roda**. O agente apresenta o plano e aguarda o "ok". Aprovação parcial (subconjunto de passos) é válida e vira o escopo efetivo.

**MUST — após aprovado**

4. **Oferecer a modalidade de execução.** O agente pergunta como rodar:
   - **Faseado e assistido** — executa por fase, **pausa entre fases** para revisão/confirmação, mostra resultado parcial e só segue com o "continuar". Default recomendado para staging sensível e para qualquer plano com passo destrutivo.
   - **De uma vez (autônomo)** — executa o plano inteiro sem parar.
5. **No modo "de uma vez":**
   - **Multiagentes para produção da execução** — paralelizar categorias independentes (ex.: `security`, `authz`, `hardening`, `pentest`, `simulated` em workers separados), respeitando dependências declaradas no plano e os limites de concorrência (§9 backpressure).
   - **Cron/watchdog de continuidade ininterrupta** — um agendador supervisiona a execução e a **retoma de checkpoint até concluir**, sem exigir intervenção manual se um worker cair. A "conclusão" é condição de parada explícita: todos os passos aprovados terminaram, **ou** uma falha bloqueante escalou para o humano. Checkpoints são **idempotentes** (§19) e a retomada não reexecuta passo já concluído.

**MUST — segurança do fluxo (herda do resto do §22)**

- Passo destrutivo (`service-kill`, `db-disconnect`, drop, mutação em massa) **só roda se constava no plano aprovado** e com o gate de ambiente ligado (ex.: `<PROJECT>_CHAOS_ALLOW=1` — §22.3). Aprovação do plano não dispensa o gate.
- Modo autônomo "de uma vez" roda por default só em `dev`/`staging`. Produção exige confirmação adicional explícita no momento da execução (alinha com §22.8 — alvo autorizado).
- **Sem retry infinito** no watchdog: a continuidade tem limite de tentativas por passo, backoff, e escala para humano ao estourar (§9, §18). "Ininterrupto até finalizar" é retomar até concluir, **não** repetir pra sempre.

**VETADO**

- Pular o plano ou a aprovação "pra ir mais rápido" — é macaquice na linha da §37 (atalho que troca segurança por velocidade). Q.A. sem plano aprovado registrado **não roda**.

> O plano aprovado é o contrato da execução. Multiagente e cron aceleram o *como*, nunca dispensam o *o quê foi autorizado*.

---

---

## 23. Makefile Padrão

```bash
make dev              # ambiente local
make test             # unit + integration
make test-unit
make test-integration
make lint
make fmt
make build
make run
make docker
make migrate
make docs             # gera/valida openapi
make security-scan    # sast + sca local
make smoketest
make pentest-light
make ci               # tudo que o CI roda
make clean
```

---

---
