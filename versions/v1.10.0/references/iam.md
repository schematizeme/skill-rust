# IAM — Identidade e Autorização da casa (piso inegociável, angle Rust)

Piso normativo de **identidade, autenticação e autorização** da casa, especializado para
**backend Rust** (o serviço de auth é um microserviço Rust; Go entra só como auxiliar,
decisão em ADR). **Todo projeto começa com um IAM robusto por desenho** — segurança é
inegociável. A base agnóstica vive na `schematize-engineering` (`references/iam.md`); aqui
ela ganha a topologia de serviço, as crates e os padrões async/Tokio da casa.

Princípios-âncora: **separar identidade de autorização**; **nunca menos de 2 fatores**
(senha + Email OTP **já conta** como 2FA baseline — fator forte é incentivado e just-in-time,
**nunca muro pré-login**); **força adaptável ao risco** (2FA→3FA + negação deceptiva sob
suspeita); **recuperação tão forte quanto o login**; **deny-by-default**; **enforcement sempre
no servidor**. O buraco clássico é ter 2FA no login e um reset por 1 email que passa por
cima — aqui isso é vetado.

## 1. Topologia — auth é uma APLICAÇÃO SEPARADA (microserviço Rust)

- **A autenticação é um serviço próprio, com link próprio e front próprio**, servido em
  **`auth.<domain>`**. **VETADO** apensar o auth ao escopo principal como monolith.
- **Microserviço de auth em Rust** (`<projeto>_auth_rs`, ex.: **axum** ou **actix-web** sobre
  Tokio) + **front de auth** próprio (`<projeto>_authfront`), com **repo, deploy, user Linux e
  systemd/container isolados** por conta própria (casa com o isolamento por app do `ops.md`
  §3). Comprometer o app principal **não** compromete o IdP.
- **O app principal (e todo cliente) delega ao auth por OIDC/OAuth2.1 + PKCE:** redireciona
  pra `auth.<domain>`, recebe tokens de volta. O `<projeto>_auth_rs` é o **IdP da casa**
  (self-hosted, consumido por N apps).
- **Chave de assinatura de token vive SÓ no `<projeto>_auth_rs`.** Ele expõe um endpoint
  **JWKS** (`/.well-known/jwks.json`); consumidores validam por **JWKS público** (ex.:
  `jsonwebtoken` com a chave buscada e cacheada), **nunca guardam a chave privada**. Segredos
  vêm do seed global / secret manager, nunca hardcoded (casa com os pisos de segurança).

## 2. Modelo de identidade

- **ID interno imutável e opaco** (ULID/UUIDv7 — crates `ulid`/`uuid`) é o `sub`. **Email e
  telefone NUNCA são ID** — são *identificadores* ligados ao usuário, cada um com estado de
  verificação. No schema (`sqlx`/Postgres): tabela `users(id)` + tabela `identifiers(user_id,
  kind, value, verified_at)` 1..N, **nunca** email como PK.
- **Identificadores 1..N por usuário** (emails, telefones, identidades SSO, passkeys, apps,
  chaves FIDO2). **Ter mais de um email é incentivado** (resiliência a brick de provedor).
- **Identificador só vale verificado** — não loga nem recupera sem verificação (`verified_at`
  não-nulo).
- **SSO nunca é ponto único de falha:** cadastro via SSO **força ≥1 fator de recuperação
  local** (email de recuperação + códigos de backup), pra provedor banido ≠ conta perdida.
- **Account-linking explícito:** SSO chegando com email já verificado em outra conta →
  linkar vs. bloquear **com confirmação** (anti-takeover). Nunca linkar por email não
  verificado.
- **Nudge de email secundário (anti-brick):** com só 1 email, a UI **sugere adicionar um
  secundário**. **Detecta o provedor** do atual (gmail / hotmail-outlook / yahoo /
  próprio-corporativo) e **recomenda que o secundário seja preferencialmente de OUTRO
  provedor**, com um **"i" de tooltip no hover**: *"Um segundo email, de preferência em outro
  provedor, garante que você não perca o acesso caso perca acesso a este email."* Sugestão,
  não obrigação.

## 3. Fatores e níveis de garantia (AAL — NIST 800-63B)

Classificar a **força** de cada fator permite "email sempre disponível" sem abrir mão de
segurança: operação sensível exige fator forte; email/SMS servem de fallback.

| Tier | Fatores | Uso |
|---|---|---|
| **Alto (phishing-resistant)** | **Passkey/WebAuthn (núcleo)**, chave FIDO2, push aprovado no app | Ops sensíveis: trocar fator, admin, cross-tenant, billing, recuperação |
| **Médio** | TOTP (app autenticador), senha + posse | Login + 2º fator |
| **Baixo (fallback)** | **Email OTP (Resend)**, **SMS/voz (Twilio)** | **É o 2º fator baseline da conta** (senha+OTP = 2FA); sempre disponível; **não** autoriza ação sensível sozinho (aí exige step-up forte) |

- **Passkey/WebAuthn é núcleo** (não roadmap): já é "2 fatores num" (dispositivo +
  biometria), phishing-resistant. Em Rust, use a crate **`webauthn-rs`** para registro/asserção.
- **Email OTP (Resend) ligado por padrão, inclusive em HML** — só o operador desliga.
- **Twilio por padrão** para verificação de telefone e 2FA por SMS/voz.
- **Provedores plugáveis por trait:** `EmailProvider` (Resend default), `SmsProvider` (Twilio
  default), `PushProvider` são **traits** com impls selecionadas por config (DI), trocáveis
  sem tocar no core. Chamadas de rede são `async` e resilientes (timeout, retry+backoff+jitter,
  store-and-forward em falha — casa com `dados-eventos.md`).
- **Senha por padrão, opcional por escolha:** o usuário **cria senha no cadastro** (padrão
  cultural; **argon2id** via crate **`argon2`** + verificação contra base de vazadas/HIBP por
  k-anonymity), mas o **seletor de modos de autenticação permite marcá-la como opcional** e
  viver de passkey/OTP/app. O hashing argon2id roda em **`spawn_blocking`** (CPU-bound, nunca
  bloqueia o runtime Tokio — casa com `async-concorrencia.md`).
- **2FA por desenho desde o cadastro — senha + Email OTP JÁ é 2FA (fraco, porém válido):**
  a conta **nasce com dois fatores obrigatórios** (senha + código no email verificado,
  always-on) e **já é segura para o baseline**. **VETADO** tratar senha+OTP como "sem 2FA" e
  **barrar o login** até enrolar um fator forte — é o **círculo infinito**. O middleware
  **libera o acesso baseline** (sessão de AAL médio); **não** exige AAL alto em toda rota.
- **Fator forte é INCENTIVADO + just-in-time, nunca muro pré-login:** app OTP / passkey / chave
  são **nudge** e **exigidos só na operação sensível** (o PEP checa o AAL mínimo **por rota
  sensível** e dispara **step-up**) e **escalados sob risco** (§9). Enrolar um fator forte usa o
  Email OTP como verificação (Y≠X, §4): sem deadlock. A ausência de fator forte **degrada o
  sensível** (`403 step_up_required`), não **bloqueia o baseline**.

## 3.1 Efeito externo fora de prd — o guard mora DENTRO do provider (recorte Rust)

O Email OTP é always-on (§3): é **daqui** que sai e-mail em todo ambiente. A normativa completa
(as 4 camadas, o DNS, a exceção com ADR) está na **schematize-engineering**,
`references/efeitos-externos.md` — este é o **recorte Rust**. Regra: fora de `prd`, **nenhum
e-mail/SMS/push/webhook chega em ninguém**, por construção. Um laço de teste que cria 5.000
contas dispara 5.000 OTPs; endereço sintético vira **hard bounce**/spam trap, e bounce acima do
limiar **queima o IP e o domínio** — derrubando o transacional de prd, **inclusive o OTP de
login deste próprio IAM**. Semanas de warm-up, utilidade zero, sem undo.

**Piso Rust, em quatro linhas:**

1. **`EmailProvider` já é trait** (§3) — o guard é um **wrapper genérico** sobre ela, e o
   provider real fora de prd é o **`SinkProvider`** (Mailpit/`tracing`).
2. O guard **não mora no caso de uso** (chamador esquece; teste chama o SDK direto): ele
   embrulha o provider **na composição**, uma vez, e o resto do sistema só vê `dyn EmailProvider`.
3. Ele devolve **`Result<(), MailError>`** com `thiserror` — recusa é **`Err` tipado**, nunca
   `tracing::warn!` + `Ok(())`. **Sem `unwrap()`/`expect()`** em caminho de produção.
4. **Cap por execução** com `AtomicUsize` (`MAIL_MAX_PER_RUN`, default 50) — os 5.000 só
   existiram porque **nada estava contando**.

**Endereço sintético:** `<papel>+<run-id>-<n>@test.<domain>` (ou TLD reservado `.test`/
`.invalid`/`.example`). **VETADO** em fixture/seed/persona/demo/carga: `@gmail.com`,
`@hotmail.com`, domínio de cliente/terceiro, e-mail de pessoa real (**inclusive o seu**) e o
domínio de **produção**. Chave de não-prd é **sandbox**, nunca a de prd.

### O erro tipado e o ambiente fail-closed — `src/platform/mail`

```rust
//! Borda de saída de e-mail (adaptador DDD). O core depende da trait; quem sabe
//! falar com Resend/Mailpit é este módulo.

use std::fmt;
use std::sync::atomic::{AtomicUsize, Ordering};
use async_trait::async_trait;
use thiserror::Error;

/// O que o domínio pede pra enviar — sem tipo de SDK vazando pra dentro.
#[derive(Debug, Clone)]
pub struct Message {
    pub to: String,
    pub subject: String,
    pub body: String,
}

/// Erro de envio. Variantes distintas porque o chamador (e o teste) precisa
/// distinguir "bloqueado pelo piso" de "provedor caiu" — não por string.
#[derive(Debug, Error)]
pub enum MailError {
    /// Destinatário fora do domínio de teste com `env != prd`. A mensagem ENSINA o
    /// caminho certo (piso de mensagem acionável) e diz que nada foi enviado.
    #[error(
        "destinatário externo bloqueado em env={env}: {redacted}. \
         Use <papel>+<run-id>-<n>@{hint} (null MX) ou registre o ADR de exceção \
         (as 5 condições). Nada foi enviado."
    )]
    ExternalRecipientBlocked { env: Env, redacted: String, hint: String },

    /// Circuit breaker do cap por execução — o que teria contido os 5.000.
    #[error("cap de envio estourado: tentativa {attempt}, teto {max} (MAIL_MAX_PER_RUN). Nada foi enviado.")]
    RunCapExceeded { attempt: usize, max: usize },

    /// Falha real do provedor. O adaptador converte o erro do SDK em String pra
    /// não acoplar o core (nem o teste) ao `reqwest`.
    #[error("falha ao entregar ao provedor: {0}")]
    Transport(String),
}

/// Ambiente resolvido.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Env {
    Dev,
    Hml,
    Prd,
}

impl Env {
    /// FAIL-CLOSED: variável ausente, vazia ou desconhecida cai em `Dev` (o modo
    /// seguro). "Não declarei o ambiente" NUNCA pode significar produção.
    pub fn from_raw(raw: Option<&str>) -> Self {
        match raw.map(|s| s.trim().to_ascii_lowercase()).as_deref() {
            Some("prd" | "prod" | "production") => Env::Prd,
            Some("hml" | "stg" | "staging") => Env::Hml,
            _ => Env::Dev,
        }
    }
}

impl fmt::Display for Env {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(match self {
            Env::Dev => "dev",
            Env::Hml => "hml",
            Env::Prd => "prd",
        })
    }
}

/// A porta. `async_trait` porque precisamos de `dyn EmailProvider` na composição
/// (`async fn` nativo em trait ainda não é dyn-compatible).
#[async_trait]
pub trait EmailProvider: Send + Sync {
    async fn send(&self, msg: &Message) -> Result<(), MailError>;
}

/// Esconde a parte local do endereço: PII nunca vai crua pra log nem pra erro (LGPD).
fn redact(addr: &str) -> String {
    match addr.rsplit_once('@') {
        Some((_, domain)) if !domain.is_empty() => format!("***@{domain}"),
        _ => "***".to_string(),
    }
}
```

### O sink (default fora de prd) e o provider real

```rust
/// Default fora de prd: a mensagem NÃO sai da máquina. Em dev/hml aponte pro
/// Mailpit (SMTP :1025, API HTTP :8025) quando o teste precisar LER a caixa; o log
/// estruturado basta quando só precisa provar que houve envio.
pub struct SinkProvider;

#[async_trait]
impl EmailProvider for SinkProvider {
    async fn send(&self, msg: &Message) -> Result<(), MailError> {
        tracing::info!(
            provider = "sink",
            to = %redact(&msg.to),
            subject = %msg.subject,
            "mail.sink"
        );
        Ok(())
    }
}

/// Provider real. Só é construído na composição quando `env == Prd` (ou sob a
/// exceção com ADR); em não-prd a chave, se existir, é a **sandbox**.
pub struct ResendProvider {
    http: reqwest::Client,
    api_key: String,
    from: String,
}

impl ResendProvider {
    pub fn new(api_key: String, from: String) -> Self {
        Self { http: reqwest::Client::new(), api_key, from }
    }
}

#[async_trait]
impl EmailProvider for ResendProvider {
    async fn send(&self, msg: &Message) -> Result<(), MailError> {
        let resp = self
            .http
            .post("https://api.resend.com/emails")
            .bearer_auth(&self.api_key)
            .json(&serde_json::json!({
                "from": self.from,
                "to": [&msg.to],
                "subject": msg.subject,
                "html": msg.body,
            }))
            .send()
            .await
            .map_err(|e| MailError::Transport(e.to_string()))?;

        let status = resp.status();
        if !status.is_success() {
            return Err(MailError::Transport(format!("resend respondeu {status}")));
        }
        Ok(())
    }
}
```

### O guard — wrapper genérico, deny-by-default, cap por `AtomicUsize`

```rust
/// Embrulha um `EmailProvider` e é o ÚNICO ponto que decide se a mensagem pode
/// sair. Genérico em `P`: sem `dyn` no caminho quente e vale pra qualquer impl.
/// O campo `inner` é privado — não dá pra "desembrulhar" por engano.
pub struct Guarded<P: EmailProvider> {
    inner: P,
    env: Env,
    test_domains: Vec<String>,
    max_per_run: usize,
    sent: AtomicUsize,
}

impl<P: EmailProvider> Guarded<P> {
    /// Aplique em TODOS os ambientes: em prd o guard de domínio é no-op, mas o CAP
    /// continua valendo (laço maluco também existe em prd). `max_per_run == 0` cai
    /// no default 50 — fail-closed: "não configurei" não vira "ilimitado".
    pub fn new(inner: P, env: Env, test_domains: Vec<String>, max_per_run: usize) -> Self {
        Self {
            inner,
            env,
            test_domains: test_domains
                .iter()
                .map(|d| d.trim().to_ascii_lowercase())
                .collect(),
            max_per_run: if max_per_run == 0 { 50 } else { max_per_run },
            sent: AtomicUsize::new(0),
        }
    }

    /// Domínio de teste exato ou subdomínio dele (`test.acme.com` cobre `a.test.acme.com`).
    fn is_test_domain(&self, domain: &str) -> bool {
        self.test_domains.iter().any(|t| {
            domain == t || domain.strip_suffix(t.as_str()).is_some_and(|p| p.ends_with('.'))
        })
    }

    /// O guard propriamente dito: em prd entrega de verdade; fora de prd só passa
    /// endereço no domínio de teste. Endereço malformado também é recusa — deny-by-default.
    fn assert_deliverable(&self, to: &str) -> Result<(), MailError> {
        if self.env == Env::Prd {
            return Ok(());
        }
        let domain = to.rsplit_once('@').map(|(_, d)| d.to_ascii_lowercase());
        match domain {
            Some(d) if self.is_test_domain(&d) => Ok(()),
            _ => Err(MailError::ExternalRecipientBlocked {
                env: self.env,
                redacted: redact(to),
                hint: self
                    .test_domains
                    .first()
                    .cloned()
                    .unwrap_or_else(|| "test.<domain>".to_string()),
            }),
        }
    }
}

#[async_trait]
impl<P: EmailProvider> EmailProvider for Guarded<P> {
    /// Ordem: deny-by-default no destinatário, cap por execução, e só então delega.
    /// Qualquer recusa devolve `Err` — nunca `warn!` + `Ok(())`, nunca no-op silencioso.
    async fn send(&self, msg: &Message) -> Result<(), MailError> {
        self.assert_deliverable(&msg.to)?;

        let attempt = self.sent.fetch_add(1, Ordering::Relaxed) + 1;
        if attempt > self.max_per_run {
            return Err(MailError::RunCapExceeded { attempt, max: self.max_per_run });
        }

        self.inner.send(msg).await
    }
}
```

### Seleção por ambiente na composição, nunca no chamador

```rust
/// Config de e-mail vinda do seed do ambiente (`/<app>/.env`, ver `ops.md`).
pub struct MailConfig {
    pub env: Env,
    pub resend_api_key: String,
    pub from: String,
    pub test_domains: Vec<String>,
    pub max_per_run: usize,
}

/// A escolha do provider acontece UMA vez, aqui — o resto do sistema recebe
/// `Arc<dyn EmailProvider>` por injeção e não conhece ambiente.
pub fn build_email_provider(cfg: MailConfig) -> Arc<dyn EmailProvider> {
    match cfg.env {
        Env::Prd => Arc::new(Guarded::new(
            ResendProvider::new(cfg.resend_api_key, cfg.from),
            cfg.env,
            cfg.test_domains,
            cfg.max_per_run,
        )),
        // DEFAULT fora de prd: sink. Nem existe caminho pro provider real aqui.
        _ => Arc::new(Guarded::new(SinkProvider, cfg.env, cfg.test_domains, cfg.max_per_run)),
    }
}
```

O caso de uso do OTP recebe `Arc<dyn EmailProvider>` e **trata o `Result`** (`?` até a borda
HTTP). Não existe parâmetro `force`/`skip_guard`: bypass por argumento é a porta que sempre
acaba aberta em produção.

### O teste que vê o vermelho

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// Conta chamadas: prova que o provider real NÃO foi acionado.
    #[derive(Default)]
    struct SpyProvider {
        calls: AtomicUsize,
    }

    #[async_trait]
    impl EmailProvider for SpyProvider {
        async fn send(&self, _msg: &Message) -> Result<(), MailError> {
            self.calls.fetch_add(1, Ordering::Relaxed);
            Ok(())
        }
    }

    fn msg(to: &str) -> Message {
        Message { to: to.into(), subject: "otp".into(), body: "123456".into() }
    }

    /// Prova o piso: em hml, e-mail pra caixa real é ERRO e nada sai. Se este teste
    /// ficar verde por acidente (guard removido), o próximo laço de carga manda
    /// 5.000 e-mails de verdade.
    #[tokio::test]
    async fn guard_recusa_dominio_externo() {
        let guarded = Guarded::new(
            SpyProvider::default(),
            Env::Hml,
            vec!["test.exemplo.com.br".into()],
            50,
        );

        let res = guarded.send(&msg("alguem@gmail.com")).await;

        assert!(
            matches!(&res, Err(MailError::ExternalRecipientBlocked { .. })),
            "esperava ExternalRecipientBlocked; veio {res:?}"
        );
        assert_eq!(
            guarded.inner.calls.load(Ordering::Relaxed),
            0,
            "provider real não pode ter sido chamado"
        );
    }

    /// Prova o circuit breaker: o 3º envio com cap=2 falha, mesmo em domínio de teste.
    #[tokio::test]
    async fn guard_aborta_ao_estourar_o_cap() {
        let guarded = Guarded::new(
            SpyProvider::default(),
            Env::Hml,
            vec!["test.exemplo.com.br".into()],
            2,
        );
        let m = msg("qa+run42-1@test.exemplo.com.br");

        for i in 1..=2 {
            assert!(guarded.send(&m).await.is_ok(), "envio {i} dentro do cap falhou");
        }

        let res = guarded.send(&m).await;
        assert!(
            matches!(&res, Err(MailError::RunCapExceeded { attempt: 3, max: 2 })),
            "esperava RunCapExceeded no 3º envio; veio {res:?}"
        );
        assert_eq!(guarded.inner.calls.load(Ordering::Relaxed), 2);
    }
}
```

O mesmo desenho vale para `SmsProvider` (Twilio), `PushProvider` e qualquer webhook de
terceiro: **trait + sink default + `Guarded<P>` + cap**. Entregar de verdade fora de prd exige
**as cinco** condições da normativa (ADR aceito, allowlist ≤5 endereços da casa, cap, janela
com expiração, subdomínio de envio separado) — faltou uma, não roda.

## 4. Fluxos

**Onboarding:** cita um email → **verifica** → **cria senha** (ou já passkey/app) → **pronto:
2FA baseline (senha + Email OTP) e acesso baseline pleno**. Só **depois**, já dentro, o sistema
**sugere** (nudge, não obriga) reforçar: 2º email de backup + fator forte. **Nunca se barra o
acesso por não ter fator forte** — ele é pedido *just-in-time* na 1ª ação sensível (step-up)
ou sob risco (§9).

**Login:** (1) sem app de 2FA ativo → **OTP por email** (mesmo sem nada habilitado); (2)
com app → **pergunta app ou email**; (3) com vários fatores (passkey/telefone/app) →
**lista todos e o usuário escolhe** qual usar. App = push-approval ou TOTP.

**Gestão de fator — invariante único:**
> **Para mutar o fator X, apresente um fator Y ≠ X, no maior AAL disponível.**
- Desativar/trocar **app** → verifica por **email** (ou outro ≠ app).
- Trocar/adicionar **email complementar** → exige o **app**.
- Add/remover **chave ou telefone** → mesmo princípio, **lista qual usar**.
- Toda mudança **notifica todos os canais verificados**; remover o **último fator forte** =
  **ação com atraso cancelável** (janela pra abortar se for ataque — job agendado que só
  efetiva após a janela, cancelável).

**Recuperação:** múltiplos caminhos independentes (vários emails, códigos de backup
offline, telefone). **Força ≥ login** (2 fatores ou processo com atraso + revisão),
rate-limit agressivo, tudo auditado. **Reset nunca é bypass de 1 fator.**

## 5. Multi-tenant + RBAC/ABAC — motor ReBAC (estilo Zanzibar)

- **Identidade global, autorização por tenant:** um usuário (1 identidade) pertence a **N
  tenants** via **membership**, com papéis **diferentes por tenant**.
- **Motor de relação (ReBAC), ex. OpenFGA/SpiceDB** — hand-rolar authz em Rust é onde vazam
  privilégios. Autorização em **tuplas** `(objeto, relação, usuário/userset)`:
  - `tenant:acme#member@user:01H…`
  - `role:acme/finance-approver#assignee@user:01H…`
  - `invoice:987#parent@tenant:acme` (recurso parenteado ao tenant)
  - permissão computada por *relation rewrite* (member do tenant **E** assignee de papel
    que concede a permissão).
- **Escrita de tuplas** (membership, atribuição de papel, parentesco de recurso) acontece no
  serviço de domínio via cliente do motor (gRPC/HTTP `async`), **transacional com o efeito de
  negócio** (outbox quando o motor é externo — nunca dual-write solto).
- **RBAC granular:** permissão = **`recurso:ação`** (`invoice:approve`, `user:invite`);
  papéis-padrão (owner/admin/member/viewer) **+ papéis 100% customizados e granulares por
  tenant** (viram relations/usersets). Deve ser possível criar cargos extremamente granulares
  e atribuí-los.
- **ABAC por cima:** condições sobre atributos (usuário/recurso/contexto — hora, IP, risco)
  via **conditional/contextual tuples** (ex.: aprova invoice < 10k do próprio setor).
- **PDP/PEP separados:** PDP = **Check API do motor**; **PEP = uma `tower::Layer`/middleware
  (axum) ou extractor** em cada serviço que chama o Check antes do handler. **Deny-by-default**,
  enforcement **server-side**, **todo endpoint mapeia 1 permissão** (`recurso:ação`). O
  `tenant_id`/`sub`/role saem **do token verificado**, nunca do body/header do cliente.
- **Token fino:** carrega `sub`/tenant/sessão/AAL — **sem** a lista de permissões (evita
  authz stale em token longo); decisão consultada no motor e cacheada com TTL curto.
- **Toda decisão de authz é logada** (quem / o quê / allow-deny / política), com `trace_id`
  (casa com a observabilidade LGTM+) — auditoria + rotina de testes.

## 6. Sessão, multi-dispositivo e logout

- **Multi-dispositivo de 1ª classe:** N sessões simultâneas por usuário, cada uma atada a
  um **dispositivo** (fingerprint + rótulo amigável "Chrome no Windows", IP/geo, último
  uso). Nenhuma sessão derruba a outra. Registro de sessão em Postgres (`sqlx`) e/ou Redis.
- **View de dispositivos/sessões:** lista os ativos e **permite remover um** (revoga a
  sessão daquele device), além de **"sair de todos"**.
- **Sessão longa por padrão (fim do "15 min e é chutado"):** o access token continua curto
  (ex.: 15 min) **mas com refresh silencioso** — para o usuário, a sessão **persiste 7 dias
  por padrão**. No login, **pergunta se o dispositivo é confiável**; se sim, **90 dias**.
  Ops sensíveis ainda pedem **step-up fresco** em AAL alto — sessão longa não enfraquece.
- **Refresh rotativo com detecção de reuso** (reusou → revoga a **família** inteira). A
  família e o `jti` são rastreados server-side.
- **Botão "Sair" bem visível → kill IRREVERSÍVEL da sessão:** não basta apagar o cookie —
  **revoga o refresh token (e a família), apaga o registro de sessão server-side, joga o
  `jti` na denylist (Redis/DB) até expirar e desassocia o push token do device**. Depois do
  logout, aquela sessão é irrecuperável: nem replay, nem refresh, nem "voltar o cookie"
  reativa. O PEP checa a denylist de `jti` a cada request.
- Cookies **`HttpOnly` + `Secure` + `SameSite`**; token nunca em `localStorage`.

## 7. Migração de auth legado — PRIORIDADE 0

Existe auth no padrão antigo → **portar pra este IAM é prioridade máxima** (segurança
inegociável; pode gastar o que precisar). Estratégia **strangler-fig**: dual-run, **re-hash
preguiçoso** no login (ex.: valida no algoritmo antigo e re-grava em **argon2id** na hora),
mapeia registros legados → modelo novo (dedupe de emails, cunha IDs internos ULID/UUIDv7),
**ativa o Email OTP always-on como 2º fator baseline** (a conta migrada já entra em 2FA sem
muro) e **incentiva enrolar fator forte** (step-up para sensível), **revoga sessões legadas** e
**nunca confia na authz legada** (re-deriva pelas tuplas do ReBAC). O auth migrado nasce já
como **microserviço Rust separado** (§1). Legado PHP/Node de auth **não** vira base de código
nova — é substituído; código Node/PHP restante segue a regra de migração da casa.

## 8. Rotina agressiva de testes (detalhe na schematize-pentest)

Suíte adversarial **contínua** (CI + agendada, fixtures multi-tenant, saída
machine-readable, **gate que trava** em qualquer vazamento):
- **Cross-tenant (BOLA/IDOR):** token do tenant B → IDs do tenant A = 403/404; fuzz de IDs.
- **Priv-esc (BFLA):** papel baixo → ação de papel alto (horizontal e vertical).
- **Matriz persona × endpoint** exaustiva.
- **Abuso de fluxo:** bypass de 2FA, reset pulando 2FA, brute-force/rate-limit de OTP,
  replay de token, reuso de refresh, JWT `alg=none`/kid trocado, session fixation, adulteração
  de asserção SSO, IDOR na gestão de identificadores, bypass de step-up, mass-assignment de
  papel, **logout que não invalidou de verdade** (sessão recuperável).

## 9. Autenticação adaptativa por risco (robusta) + transversais

A resposta ao login **varia com o risco calculado** — é o que torna a conta difícil de tomar
sem chatear o legítimo:
- **Log de sessões/tentativas:** IP/ASN+reputação, device fingerprint, geo, UA, horário,
  resultado e **score de risco** — na view de sessões (§6) e em audit log imutável.
- **Score por tentativa:** IP suspeito/novo, device novo, geovelocidade impossível,
  velocity/brute, hit de honeypot. Baixo = normal; alto = escala.
- **Escalonamento por risco (2FA→3FA):** sob risco, exige **um fator a mais** — **senha →
  email → app/chave**. Contexto (não só a ação) dispara step-up (mesmo motor do §3).
- **Negação deceptiva / tarpit (falso negativo sob risco):** em contexto suspeito, mesmo com
  **senha correta** o serviço pode responder **genérico uma vez** enquanto **computa que estava
  certa** e marca "próxima passa" (já com os fatores escalados). Seguro porque: **resposta e
  tempo idênticos** ao erro real (comparação em tempo constante, mesmo path — sem oráculo);
  estado curto/escopado (conta+IP+device, TTL curto, nunca lockout do legítimo); **soma-se** ao
  3FA; tudo logado.
- **Honeypot:** contas/campos/rotas isca → qualquer interação = hostil (score alto, tarpit,
  alerta); nunca serve real.
- **Anti-automação:** rate-limit + **backoff exponencial** + **lockout progressivo por
  conta+IP**; OTP curto/single-use, `jti` na denylist. Notifica login suspeito + "não fui eu".

### Transversais (sempre)
- **Audit log imutável** de toda decisão authn/authz e mudança de credencial — alimenta a
  forense e os testes (liga com a observabilidade LGTM+ da casa; `trace_id` propagado).
- **Padrões:** OIDC/OAuth2.1 + PKCE; WebAuthn/FIDO2 (`webauthn-rs`); AALs NIST 800-63B; SCIM
  (roadmap enterprise); FAPI2 se fintech.
- **Crates de apoio:** `argon2` (hash), `jsonwebtoken`/`josekit` (JWT/JWKS), `webauthn-rs`
  (passkey), `totp-rs` (TOTP), `ulid`/`uuid` (ID), `sqlx` (persistência), cliente do motor
  ReBAC (OpenFGA/SpiceDB). Nenhuma chave privada fora do `<projeto>_auth_rs`.

## Roadmap de fases
- **F0** Núcleo de identidade (ID imutável, N identificadores, verificação, Resend/Twilio
  plugáveis por trait, email OTP always-on) — já como **microserviço Rust separado** em
  `auth.<domain>`.
- **F1** 2FA baseline por desenho (senha + Email OTP, sem muro pré-login) + fluxos (TOTP/push,
  **passkey** via `webauthn-rs`, escolha de método, invariante de troca, **nudge** de fator
  forte, step-up just-in-time, **risk engine adaptativo**: score, 2FA→3FA, negação deceptiva/
  tarpit, honeypot).
- **F2** Multi-tenant + **ReBAC** (membership, papéis granulares, PDP/PEP como `tower` layer,
  deny-default, token fino, audit).
- **F3** Recuperação resiliente (múltiplos caminhos, força ≥ login, SSO com recuperação
  local, atraso cancelável, nudge de email secundário).
- **F4** App de 1ª classe + multi-dispositivo (OIDC/PKCE nativo, push-approval, view de
  remover dispositivos, sessão 7d/90d confiável, logout irreversível).
- **F5** Migração de legado (prioridade 0 quando aplicável).
- **F6** Rotina agressiva de testes (cross-tenant, priv-esc, abuso de fluxo — CI + agendada).
- **Roadmap+** chave FIDO2 dedicada, SCIM, FAPI2, trusted contacts.

## Checklist (entra na Definition of Done quando o projeto tem auth)
- [ ] Auth é **app separada** (`<projeto>_auth_rs` + front próprio em `auth.<domain>`, isolados) — não monolith.
- [ ] **ID interno imutável** (ULID/UUIDv7); email/telefone não são ID; múltiplos emails suportados.
- [ ] **2FA baseline por desenho** (senha + Email OTP = 2FA desde o cadastro); fator forte é **nudge + just-in-time (step-up)**, **NUNCA muro pré-login** (o PEP libera o baseline, exige AAL alto só por rota sensível); passkey no núcleo (`webauthn-rs`); email OTP always-on; Twilio.
- [ ] **Risk engine adaptativo:** log de sessões/tentativas + score (IP/device/geo/velocity/honeypot); **2FA→3FA** sob risco; **negação deceptiva/tarpit** (falso negativo, resposta idêntica ao erro real em tempo constante, "próxima passa" curta/escopada); notifica login suspeito.
- [ ] Invariante de troca de fator (Y≠X, maior AAL); recuperação ≥ login; SSO com recuperação local.
- [ ] **Multi-tenant + RBAC/ABAC** (ReBAC), deny-default, PDP/PEP (`tower` layer), enforcement server-side, token fino.
- [ ] Multi-dispositivo + view de remover; **sessão 7d/90d**; **logout irreversível** (revoga refresh+família, `jti` na denylist, não só cookie).
- [ ] argon2id (crate `argon2`, em `spawn_blocking`); chave de assinatura só no auth (JWKS público).
- [ ] Audit log de authn/authz; risk engine/rate-limit; migração de legado tratada como prioridade 0.
- [ ] Rotina agressiva de testes cross-tenant/priv-esc no CI (schematize-pentest).
- [ ] **Efeito externo fora de prd (§3.1):** `EmailProvider`/`SmsProvider` com **sink default** fora de `prd`, **guard `Guarded<P>` DENTRO do provider** devolvendo `Result<_, MailError>` (`ExternalRecipientBlocked`), **cap por `AtomicUsize`**, endereço sintético só em `test.<domain>` (null MX) e chave **sandbox** — provado por `guard_recusa_dominio_externo` (espera o `Err`) e `guard_aborta_ao_estourar_o_cap`.
</content>
</invoke>
