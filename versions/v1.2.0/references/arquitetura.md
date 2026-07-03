# Arquitetura, Camadas, Repositórios e Linguagens

> Parte da skill **schematize-rust**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 2. Estrutura de Repositórios
- 3. Linguagens
- 4. Arquitetura
- 5. Estrutura de Pastas
- 6. Complexidade e Tamanho
- 7. Dependências Internas e Shared Libraries
- 8. CQRS e Padrões de Aplicação

---

## 2. Estrutura de Repositórios

**MUST**
- Um repositório = uma aplicação ou um bounded context.
- Comunicação entre serviços via HTTP, gRPC, eventos ou mensageria — nunca via banco compartilhado.
- Cada serviço é dono do seu schema.
- **Nome do repositório:** `<projeto>_<contexto>[_<lang>]` em snake_case minúsculo. `<projeto>` = slug do produto/organização; `<contexto>` = a aplicação/bounded context daquele repo (`api`, `worker`, `front`, `backoffice`, `gateway`…); `_<lang>` é sufixo **opcional** pra desambiguar linguagem (`_rs` Rust, `_go` Go, `_ts` TypeScript). Como um repo = um contexto, o nome espelha isso. Ex.: `loja_api_rs`, `loja_front`, `loja_worker_go`.
- **Independência de runtime (cada serviço é entidade à parte):** todo serviço **sobe e opera sozinho**. A indisponibilidade de qualquer outro serviço **nunca** impede o boot nem derruba este — depender de outro serviço para *iniciar/funcionar* é VETADO (nada de "o `ledger` não sobe se o `core` estiver fora"). Dependente ausente vira **degradação graciosa** (fallback, resposta parcial, enfileira e segue), nunca crash em cascata. Como não perder o dado quando a chamada falha: `references/dados-eventos.md` (§18).
- **`<projeto>_ops` (control plane de desenvolvimento):** todo sistema multi-repo tem um repo **`<projeto>_ops`** — a ferramenta de operação do workspace, rodada por dev/agente e **fora do runtime do produto**. Faz bootstrap/instalação, update, manutenção, troubleshooting e roda os testes unitários/debug **através de todos os repos** (clona, sobe/para, migra, semeia e testa cada serviço). Não é microserviço nem é deployado com o produto; é essencial pra tocar um sistema de múltiplos repositórios. Como toda ferramenta, sobe com **observabilidade integrada** (Grafana/LGTM+, ver `references/observabilidade.md` §16).
- **Contenção no workspace (nunca sair da pasta do projeto):** o **diretório de projeto atual é o workspace**; toda aplicação/repo do sistema nasce e mora **dentro dele**. Vai criar uma aplicação nova? Crie uma **pasta pra ela dentro da pasta atual** (`./<projeto>_<contexto>/`) e trabalhe lá — **nunca** largue arquivos soltos no root pra depois **subir de diretório** (`cd ..`, `../`) e criar os outros repos fora. Num sistema multi-repo os repos são **irmãos dentro do mesmo workspace** (clonados ali pelo `<projeto>_ops`), não espalhados pela máquina. **VETADO** criar/ler/escrever fora do workspace: diretório-pai, `~`, `~/Documents`, `~/Downloads`, `/tmp` do usuário, Área de Trabalho. O agente **não sai da pasta do projeto** — nem pra vasculhar, nem pra criar — a menos que o usuário peça explicitamente.

**VETADO**
- **Aplicação monolítica que acopla múltiplos bounded contexts num só deploy/processo.** Não se cogita "começar monolito e quebrar depois" sem ADR explícito de plano e prazo de quebra. Misturar domínios de negócio "pra entregar rápido" é dívida disfarçada de produtividade.
- **Monólito distribuído** — o pior dos dois mundos: serviços separados fisicamente, mas acoplados por banco compartilhado, shared lib de domínio (§7) ou chamadas síncronas em cascata sem fronteira. Tão proibido quanto o monólito clássico.
- **Big ball of mud** — código sem fronteira de contexto, onde tudo importa tudo.

**SHOULD**
- Evitar mais de um domínio de negócio no mesmo repositório.
- Leitura cross-service por réplica read-only só com ADR e contrato documentado.

**MAY**
- *Modular monolith* (módulos com fronteira de contexto rígida, schema separado, comunicação por interface interna) **somente com ADR** que justifique estágio do produto e contenha o plano de extração. É exceção registrada, não default. Nunca usar como atalho para colar domínios.

> Não existe "MVP monolítico que vira microserviço depois" sem o ADR que prova que o depois tem data. Sem isso, "depois" é "nunca", e "nunca" é um big ball of mud em produção.

---

---

## 3. Linguagens

**Backend — apenas Rust e Go.**

| Cenário | Stack |
|---|---|
| Serviços backend, APIs, glue code, concorrência, padrão da casa | **Go** |
| Performance crítica, segurança de memória | **Rust** |

**Frontend — Node é 100% permitido (e só frontend).** Frontend baseado em Node é a stack da casa porque hoje é o melhor do mercado: **Next.js** é a stack principal, mas **Astro e outros frameworks consolidados** são permitidos. O server-side do próprio front (route handlers, server actions, BFF) faz parte do frontend e é governado pelo §13.4 e §38 (segredo só server-side, etc.). Isso vale **apenas** para frontend — **não** reabre Node como linguagem de serviço backend (ver o MUST abaixo e §3.1: backend novo em Node é proibido; o ganho marginal de tooling/npm não compensa o histórico de incidentes de segurança).

**MUST**
- Versão exata em uso fica no Anexo A.
- Não misturar linguagens dentro do **mesmo bounded context** sem ADR.
- **Backend novo só em Rust ou Go.** Nenhum serviço backend novo em Node.

**SHOULD**
- Default é **Go**; **Rust** quando performance crítica ou segurança de memória justificam. Em empate técnico, Go vence (padrão do time).
- Frameworks são bem-vindos; abstrações mágicas não. Critério: consigo entender o stack trace?

### 3.1 Node legado (backend) — migração para Rust/Go

Node como linguagem de **serviço backend** está em saída. Tudo que existe em Node backend será migrado para Rust ou Go, guiado por esforço (não big-bang) e medido **por funcionalidade do módulo**, não por linha.

**Modelo da métrica.** Um módulo tem N funcionalidades (ex.: 10 — cálculo, ABAC, CRUD, etc.). O quanto uma mudança "pesa" é a fração de funcionalidades que ela altera ou cria sobre o total do módulo. Ex.: módulo com 10 funcionalidades — refatorar 2 = 20%; refatorar 3 (ou criar ~4 novas) ≈ 30%.

**MUST**
- **Não mexer no que está feito em Node, a menos que solicitado.** Node backend funcionando fica como está até ser tocado.
- **Gatilho de extração (~30%):** quando uma mudança atingir ~30% das funcionalidades do módulo (alteradas + novas), **não cresça o Node** — extraia essa(s) funcionalidade(s) para um **módulo Rust/Go à parte** e incorpore o comportamento Node nessa nova base.
- **Extração incremental:** conforme se mexe no módulo Node ao longo do tempo, vai-se extraindo aos poucos para Rust/Go.
- **Virada dos 50%:** quando ~50% do módulo já estiver extraído/inutilizado (substituído pela versão Rust/Go), **migra-se os 50% restantes de uma vez** — encerra o módulo Node.
- **Ajuste pontual não porta.** Mudança pequena/localizada (abaixo do gatilho) é feita no próprio Node, sem portabilidade.
- Toda migração registra ADR (§27) e segue o DDD híbrido/coexistência (§4.X, §36): flag de coexistência, sem big-bang.

> Os percentuais (~30% pra extrair, ~50% pra finalizar) são os limiares da casa; ajuste por ADR se um módulo específico exigir. A regra é: parou de ser ajuste pontual, vira extração; passou da metade, termina.

### 3.2 PHP — proibido

**VETADO** — PHP não é linguagem da casa, em nenhuma camada.

- Nenhum código novo em PHP.
- Projeto existente em PHP é **migrado sumariamente** para Rust/Go (backend) — prioridade de migração, com ADR e plano. Não é "quando der"; é dívida ativa a ser zerada.

---

---

## 4. Arquitetura

**MUST — todos os projetos**
- Separação explícita de camadas: `domain`, `application`, `infrastructure`, `interface`.
- Inversão de dependência: domínio não conhece infra.
- Domínio não importa frameworks nem ORM.

**SHOULD — projetos com regra de negócio relevante**
- DDD tático (agregados, value objects, eventos de domínio).
- Arquitetura hexagonal (ports & adapters).

**MAY — CRUDs simples**
- Manter as 4 camadas, dispensar táticas DDD pesadas.

### Dependências permitidas

```
interface       → application
application     → domain
infrastructure  → domain, application
```

### Dependências proibidas

```
domain          → qualquer outra camada
domain          → frameworks, ORM, libs de IO
application     → interface
```

### Anti-Corruption Layer

**MUST** em integrações com sistemas externos: adapter dedicado em `infrastructure/external/` que traduz o modelo externo para o modelo de domínio. **Nunca** expor DTOs externos diretamente no domínio.

### 4.X DDD híbrido durante transição

Projetos legados onde código já existe sem separação de camadas **podem** adotar DDD progressivamente em vez de big-bang. Regras:

**MUST**
- Toda nova feature/refactor em código tocado segue o layout completo (`domain/`, `application/`, `infrastructure/`, `interface/`) — não introduzir mais código "flat".
- Ao mover/quebrar arquivo legado, organize já em folders DDD mesmo que internamente alguma classe ainda misture responsabilidades (ex.: service em `application/` ainda chamando SQL direto). Estrutura primeiro, inversão depois.
- Cada PR que toca arquivo híbrido **deve** mover ao menos um pedaço pra direção certa (ex.: extrair entidade pra `domain/`, mover query pra `infrastructure/repositories/`).
- ADR registrando o débito e o plano de remoção: `<project>/docs/adr/<n>-ddd-migration-<contexto>.md`.

**SHOULD**
- Manter teste de cobertura por camada (§22) durante a transição — domain começa com 0%, sobe a cada PR.
- Linter ou guard test que **rejeita imports proibidos** logo que possível (mesmo que com whitelist de exceções legadas):
  - `domain/` não importa `@nestjs/*`, `pg`, `axios`, `infrastructure/*`, `application/*`, `interface/*`.
  - `application/` não importa `interface/*`.

**MAY**
- Marcar arquivos híbridos com comment `// @ddd-hybrid` pra busca fácil e cleanup priorizado.

---

---

## 5. Estrutura de Pastas

### Node.js

```
src/
├── domain/           # entities, value-objects, services, events, repositories (interfaces)
├── application/      # use-cases, dto, commands, queries
├── infrastructure/   # persistence, messaging, external, observability
├── interface/        # http, grpc, cli
├── shared/
└── config/
tests/
```

### Go

```
cmd/<app-name>/
internal/
├── domain/
├── application/
├── infrastructure/
├── interface/
├── shared/
└── config/
tests/
```

### Rust

```
src/
├── domain/
├── application/
├── infrastructure/
├── interface/
├── shared/
└── config/
tests/
```

---

---

## 6. Complexidade e Tamanho

> **Canônico em `references/padroes-codigo.md`** (≤ 300 linhas/arquivo, uma função/unidade por arquivo, função > 300 linhas quebrada, doc-comment obrigatório com motivo/comportamento/entradas/saídas/efeitos, e `MAPA.md`). Esta seção é o recorte de arquitetura desses pisos — não duplica a regra, contextualiza.

**MUST — arquivos pequenos, micro-funções**
- **Arquivo alvo: ≤ 300 linhas.** Acima disso, o arquivo **deve ser quebrado** — extraia responsabilidades em módulos menores e a lógica em **micro-funções** com nome que explica a intenção. Não existe "arquivo de 800 linhas porque é coeso": coesão real cabe em arquivos pequenos colaborando.
- **Funções pequenas e de responsabilidade única.** Ideal ≤ 50 linhas; função grande vira micro-funções compostas. Use case: uma responsabilidade.
- Exceções (não disparam quebra): testes, migrations, código gerado, schemas/fixtures.

**MUST — toda função documentada**
- **TODA função/método tem comentário** no formato de doc da linguagem (JSDoc, GoDoc, rustdoc). O comentário declara, no mínimo:
  - **O quê** — o que a função faz, em uma linha.
  - **Onde é usada / prevista** — quem chama, em que fluxo/camada ela foi pensada pra servir (ex.: "usado pelo use-case `CreateOrder`", "handler HTTP de `/v1/checkout`"). Isto dá contexto explícito de propósito e evita função órfã.
  - Parâmetros, retorno e efeitos colaterais relevantes.
- Esse comentário é a **fonte do índice de microfunções** (§39) — escreva pensando que ele será extraído e indexado, não como enfeite.

Convenção mínima (adapte à linguagem):

```
/**
 * O quê: valida o payload de checkout e cria o pedido.
 * Onde:  use-case CreateOrder; chamado pelo handler POST /v1/checkout.
 * Efeitos: persiste em orders, publica catalog.order.created via outbox.
 */
```

**Bloqueio rígido em CI**
- Arquivo de produção > 300 linhas sem quebra (exceto as exceções acima) → bloqueia.
- Função de produção sem doc-comment → bloqueia.
- Complexidade ciclomática > 15 em função de produção.
- Aninhamento > 4 níveis.

> Linha de código é proxy ruim para complexidade — complexidade ciclomática é a métrica honesta. Mas arquivo gigante e função sem contexto são dívidas óbvias: quebre e documente antes do merge.

---

---

## 7. Dependências Internas e Shared Libraries

**MUST**
- Shared libraries são **mínimas** e com escopo claramente delimitado.
- Permitido como shared: observabilidade, autenticação/auth, primitives de infraestrutura, SDKs internos, logging, configuração.

**MUST NOT**
- Criar `commons` / `core-lib` / `platform-utils` genéricos.
- Compartilhar **lógica de domínio** entre bounded contexts.
- Compartilhar entidades de domínio. Cada contexto modela o seu.

> O caminho mais rápido pra um monólito distribuído é uma shared lib chamada `commons`.

**SHOULD**
- Shared libs versionadas com SemVer próprio.
- Breaking changes em shared lib exigem ADR.

---

---

## 8. CQRS e Padrões de Aplicação

- **Commands**: alteram estado, retornam id ou void.
- **Queries**: nunca alteram estado, otimizadas para leitura, podem usar projeções.
- CQRS **não exige** event sourcing.

---

---
