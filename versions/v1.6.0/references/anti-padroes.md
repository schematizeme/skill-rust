# Filosofia, Aplicação Universal e Anti-Padrões Vetados

> Parte da skill **schematize-rust**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 0. Como Ler
- 0.1. Aplicação Universal — Este Arquivo é Contexto Máximo
- 1. Filosofia
- 37. Anti-Padrões Vetados — "Macaquices" que Terminam Rápido e Quebram em Produção

---

## 0. Como Ler

- **MUST / Obrigatório** — regra. Desvio bloqueia merge ou exige ADR.
- **SHOULD / Recomendado** — padrão. Desvio precisa de justificativa no PR.
- **MAY / Opcional** — sugestão.
- **VETADO / Proibido** — não existe "atalho". Não se faz, não se cogita, não se "resolve depois". Burlar é incidente, não decisão técnica.

Quando este documento conflitar com a realidade do problema, **registre um ADR** explicando o desvio. Padrão sem exceção vira dogma; dogma vira dívida. **Mas itens marcados VETADO não têm ADR de exceção** — são pisos de segurança e integridade, não preferências.

Versões concretas de stacks ficam no **Anexo A**, atualizado independentemente deste corpo.

---

---

## 0.1. Aplicação Universal — Este Arquivo é Contexto Máximo

**MUST**

- Este documento é **anexado a TODO prompt / sessão / tarefa** de engenharia (humana ou assistida por IA). É **contexto pinado**, não referência opcional. Se a tarefa toca código, infra, dados, deploy ou design de sistema, este arquivo está em contexto. Sem exceção.
- Nenhuma resposta, PR, geração de código ou decisão arquitetural é válida se ignorar este documento. "Não estava no contexto" não é desculpa — **garantir o anexo é responsabilidade de quem abre a tarefa**, e a ausência dele é, por si só, motivo para parar e recarregar o contexto antes de produzir qualquer coisa.
- Em caso de conflito entre uma instrução pontual ("faz rápido", "ignora o teste", "depois a gente arruma") e este documento, **este documento vence**. Pressa não revoga regra.
- Assistentes de IA operam sob as mesmas regras dos humanos (§34) **e** sob as proibições explícitas da §37. Velocidade de geração nunca justifica violar um piso.

> Um padrão que não está no contexto na hora da decisão é um padrão que não existe. Por isso ele é pinado, não linkado.

---

---

## 1. Filosofia

Prioridades, em ordem de desempate:

1. Clareza > esperteza
2. Simplicidade > abstração antecipada
3. Manutenibilidade > velocidade pontual
4. Observabilidade > debugging manual
5. Segurança por padrão > segurança como camada final
6. Evolução incremental > big-bang
7. **Registro do que foi decidido > memória de quem decidiu** (ver §28)

**Princípios:** Clean Code, SOLID, KISS, DRY (com bom senso — duplicação acidental ≠ duplicação semântica).

**Regra suprema:** se algo aumenta acoplamento, reduz observabilidade, cria dependência desnecessária, **vaza segredo, mistura contexto, ou pula registro** ou adiciona complexidade sem benefício claro, **provavelmente está errado**.

---

---

## 37. Anti-Padrões Vetados — "Macaquices" que Terminam Rápido e Quebram em Produção

> Atalhos que parecem entregar mais rápido e na real entregam vulnerabilidade, vazamento ou dívida. **Todos VETADOS** — não admitem ADR de exceção, são pisos. Aparecem em diff humano ou de IA → o PR para. Cada item traz o **veto** e o **caminho certo**.

### Segredos e exposição

1. **Segredo no bundle do cliente.** API key privada, secret de JWT, senha de banco, service-role key, token de pagamento no código que vai pro browser, ou em `NEXT_PUBLIC_*` / `VITE_*` / `REACT_APP_*`.
   → Segredo **só server-side** (BFF, route handler, secret manager). O navegador não guarda segredo (§13.4, §38).

2. **PII / token / senha em query string ou URL.** Acaba em log de acesso, histórico do browser, header `Referer`.
   → Vai em body ou header apropriado, nunca na URL (§32, §16.1).

3. **`.env` com segredo real commitado**, ou segredo hardcoded "temporário" no código.
   → Secret manager + `.env.example` sem valores. Gitleaks no pipeline (§13).

### Injeção e execução

4. **SQL por concatenação de string** com input externo.
   → Prepared statements / query parametrizada, sempre (§10).

5. **`eval`, `Function()`, `exec`, template string em `child_process`/`os/exec`/shell** com qualquer parte vinda de input.
   → Nunca. Args separados, allowlist de comandos, bibliotecas que não invocam shell.

6. **Desabilitar verificação TLS** (`rejectUnauthorized: false`, `InsecureSkipVerify: true`, `verify=False`) pra "funcionar logo".
   → Cert válido. mTLS interno. Se o cert está errado, conserta o cert.

### Auth e autorização

7. **Auth/authz só no client** (`if (user.isAdmin)` no React decide acesso).
   → Toda decisão de acesso é **server-side** (§15). Front é UX, não controle.

8. **Confiar em `tenant_id` / `role` / `user_id` vindos do body ou header do cliente** sem validar contra o token.
   → Derivar sempre do token verificado, server-side (§15).

9. **JWT decodado sem validar** assinatura, `exp`, `aud`, `iss`, e `alg` contra allowlist (aceitar `alg: none` ou HS256 com pubkey RS256).
   → Validação completa em toda request (§14).

10. **Hash de senha fraco** — MD5, SHA1, sem salt, ou plaintext.
    → bcrypt cost ≥ 12 ou argon2id (§14).

11. **`Math.random()` (ou rand não-cripto) pra token, id de sessão, código de reset, nonce.**
    → CSPRNG: `crypto.randomBytes`, `crypto/rand` (§14).

### CORS, headers e superfície

12. **`Access-Control-Allow-Origin: *` em rota autenticada** (pior ainda com `allow-credentials`).
    → Allowlist explícita de origens (§22.3 hardening).

13. **Endpoint de debug/admin/management sem auth, ou bind em `0.0.0.0`** expondo porta interna.
    → Bind restrito, auth obrigatória, `/debug` e `/actuator` retornam 404 externamente (§22.3).

14. **Mass assignment** — dar bind do body inteiro direto na entidade, deixando passar `is_admin`, `tenant_id`, `created_at`, `password_hash`.
    → Allowlist explícita de campos aceitos por endpoint.

### Erros, tipos e qualidade

15. **Catch que engole erro** — `catch {}`, `except: pass`, `_ = err`, `.catch(() => {})`.
    → Tratar, logar com contexto e `trace_id`, propagar ou degradar de forma consciente.

16. **`// @ts-ignore`, `any`, `interface{}`, `unwrap()`/`panic`/`!` pra calar o compilador/linter.**
    → Tipar de verdade, tratar o caso de erro. Inline-ignore de regra **de segurança** (`nolint`, `eslint-disable security/*`, `# nosec`) é VETADO sem ADR.

17. **Logar request/response inteiro, headers ou body cru "pra debugar".**
    → Logar campos específicos, mascarados. Nunca PII/token/senha (§16.1).

### Testes e cobertura

18. **Pular/comentar teste pra passar o CI** — `.skip`, `t.Skip`, `xit`, `@Ignore`, comentar o `assert`.
    → Conserta o código, não silencia o teste.

19. **Baixar o threshold de cobertura ou editar o gate** pra o número fechar.
    → Cobertura é contrato (§22). Sobe escrevendo teste, não mexendo na régua.

20. **Mockar o próprio sistema sob teste** retornando sucesso fixo, dando "verde" falso.
    → Testar comportamento real; mock só nas bordas externas.

### Dados e migrations

21. **Migration sem `down`, ou destrutiva sem backup** (`DROP`/`ALTER` que perde dado).
    → Reversível, testada com rollback antes do merge (§10).

22. **Cache de resposta autenticada sem chave por usuário/tenant** — um user recebe dado do outro.
    → Chave de cache sempre segmentada por usuário e tenant (§11, §15).

### Operação e entrega

23. **Container root, `chmod 777`, `--privileged`, filesystem RW** "pra funcionar".
    → Não-root, read-only, least-privilege (§13).

24. **Dependência nova sem verificar** nome (typosquatting), manutenção, licença, e sem pin de versão (`latest`, range frouxo).
    → Pin exato, checar nome/manutenção/licença, SCA no pipeline (§13, §34).

25. **Retry infinito / sem backoff/jitter** — DoS no próprio sistema ou no terceiro.
    → Limite explícito + backoff exponencial + jitter (§9, §18).

26. **`Idempotency-Key` aceito mas ignorado** (header existe, lógica não).
    → Implementar de fato a deduplicação (§12).

27. **Dual-write** — gravar no banco e publicar no broker no mesmo fluxo, sem outbox.
    → Transactional Outbox (§9, Anexo B).

28. **Pular o archive/MD "pra ir mais rápido"** (§28).
    → Archive é parte da entrega. Sempre gerado. Tarefa sem archive não está pronta (§35).

29. **Merge direto na `main` / force push em branch protegida / pular o PR e o review.**
    → Trunk-based com PR, CI verde, CODEOWNERS (§24).

30. **Desligar rate limit, validação de payload, ou security scan "temporariamente".**
    → "Temporário" vira permanente. Não se desliga piso de segurança (§12, §13).

31. **Criar serviço backend novo em Node, ou qualquer código novo em PHP.**
    → Backend novo só em Rust/Go; Next.js segue valendo pro frontend. PHP é proibido e migra (§3). Node backend legado segue a regra dos 30% (§3.1).

32. **Serviço que não sobe / crasha porque outro serviço está fora** (acoplamento de runtime, crash em cascata) — "o `ledger` não sobe sem o `core`".
    → Cada serviço é entidade à parte: sobe e opera sozinho; dependente ausente = degradação graciosa, nunca crash (§2, §18).

33. **Falha ao notificar/chamar outro serviço que derruba o chamador ou perde o dado** sem persistir, alertar e retomar.
    → Store-and-forward: persiste o intento (outbox/fila/Redis/DB), loga com `trace_id`, alerta (Grafana), retoma com retry+backoff+jitter → DLQ + escala (§18).
34. **Criar arquivos ou repos fora da pasta do projeto** (começar largando arquivos no root e depois **subir de diretório** — `cd ..`, `../` — pra criar repos irmãos fora; ou espalhar em `~`, `~/Documents`, `~/Downloads`, `/tmp`, Área de Trabalho).
    → Aplicação nova = **pasta dentro do workspace atual** (`./<projeto>_<contexto>/`). O agente não sai da pasta do projeto (ler ou escrever) sem o usuário pedir (§2).
35. **Editar código direto no servidor** (hml/prd), ou **subir mudança direto pra hml/prd** pulando `dev local → teste local → GitHub`.
    → Servidor é **imutável por edição manual**; recebe só artefato promovido do git. Hotfix segue o mesmo fluxo, acelerado (`references/ops.md` §1).
36. **Operar o servidor por fora do `<projeto>_ops`** — `ssh` + comando ad-hoc, editar arquivo no servidor, `docker`/`kubectl`/`systemctl` na mão, script solto.
    → **100%** de install/update/config/correção passa por comando do ops. Não tem comando? **cria no ops** (`references/ops.md` §2).
37. **Instalar/subir o sistema em série** ("um serviço de cada vez", 20 min).
    → Instalação **paralela por padrão** = `nproc` (`references/ops.md` §3).
38. **Serializar a instalação pra "funcionar"**, mascarando que um serviço depende de outro pra subir.
    → Erro que só ocorre em paralelo = **serviços não independentes** (fere piso 10/6). O ops **expõe** a colisão; corrigir a independência é **prioridade máxima**. Nunca esconder com serialização (`references/ops.md` §6).
39. **Redeploy que faz patch in-place / não parte do seed** (estado acumulado, drift entre implantações).
    → Todo redeploy é **destrutivo na app**: apaga a anterior e recria um clone zerado a partir de `/<app>/.env` (`references/ops.md` §2). Idempotente e reprodutível.
40. **Config/segredo de serviço fora do seed global**, ou repos do sistema espalhados fora de `/<app>/`.
    → `/<app>/.env` é a **fonte única** de config; o ops clona os repos dentro de `/<app>/` (`references/ops.md` §2).
41. **Apagar dados persistentes num redeploy** ("destrutivo" incluindo banco/volumes), ou `ops reset` de dados em prd.
    → Destrutivo é a **aplicação, nunca os dados**: banco/volumes/uploads preservados (migration reversível); apagar dado é `ops reset` **gated a dev/hml** (`references/ops.md` §2).
42. **Dois serviços no mesmo user Linux, serviço rodando como `root`, ou criar user/unit/permissão à mão.**
    → **Um user + systemd unit hardened por serviço**, provisionado **pelo ops** (`references/ops.md` §3). Blast radius mínimo.

> Regra de bolso: se a justificativa começa com "só pra funcionar", "depois eu arrumo", ou "é mais rápido assim" e o resultado mexe em segredo, auth, dado, registro, **ou toca o servidor por fora do fluxo/ops** — **provavelmente é uma macaquice desta lista. Para e faz certo.**

---

---
