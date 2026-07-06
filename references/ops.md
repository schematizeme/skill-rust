# Operação pelo ops — ambientes, instalação e correção (control plane)

> O **`<projeto>_ops`** é a **interface única** de operação do sistema. Três invariantes desta reference, todos **INEGOCIÁVEIS**: (1) nada chega ao servidor sem passar pelo **fluxo de promoção** (dev local → teste local → GitHub → hml → prd); (2) **100%** de instalação/atualização/correção/config no servidor passa por uma **ferramenta do ops** — nunca à mão; (3) instalar é **paralelo por padrão** (= nº de cores), e se o paralelo quebra, é **bug de independência** — correção de **prioridade máxima**. Contexto do ops: `arquitetura.md` §2. Deploy/ambientes: `operacao.md` §21. Test kit do ops: `testes.md` §22.1.

## 1. Ambientes e o fluxo de promoção (nada direto no servidor)

Ambientes isolados: **dev** (local) · **hml** (homologação/staging) · **prd** (produção). O caminho de qualquer mudança é **fixo e sem atalho**:

```
desenvolvimento local  →  teste local (verde)  →  GitHub (merge)  →  hml  →  prd
```

- **Nada pula etapa.** Nunca vai direto pra hml ou prd. hml só recebe o que está no GitHub e passou no teste local; prd só recebe o que foi homologado em hml.
- **VETADO editar código direto no servidor (hml/prd).** O servidor é **imutável por edição manual** — recebe apenas **artefato promovido do git** (mesma imagem/commit SHA, §21 "artefato imutável"). "Editei direto no prd pra resolver rápido" é o anti-padrão que esta reference existe pra matar.
- **Hotfix segue o mesmo fluxo, acelerado:** branch → teste local → git → hml → prd. Urgência **não** autoriza mão no servidor.
- **Precauções concretas** (não é regra de fé, é guarda):
  - Deploy só aceita artefato com **proveniência git** (commit SHA rastreável); artefato sem origem no git é recusado.
  - Filesystem da app **read-only** em hml/prd (container não-root/read-only, §21) — não dá pra editar mesmo.
  - **Drift detection:** o ops compara o que roda no servidor com o git e **recusa/alerta** divergência (alguém mexeu à mão → incidente, não "jeitinho").
  - Acesso de escrita ao servidor é **exceção auditada** (break-glass com registro no archive), nunca o fluxo normal.

## 2. O ops é a interface única (100% das operações)

**Toda** operação sobre o servidor — instalar, subir, atualizar, configurar, migrar, corrigir, reverter, diagnosticar — passa por um **comando do `<projeto>_ops`**. **Zero** exceção:

- **Proibido** `ssh servidor` + comando ad-hoc, editar arquivo no servidor, `docker`/`kubectl`/`systemctl` na mão, script solto de uma vez. Se **não existe** comando de ops pra aquilo, **cria o comando no ops** — não faz por fora. O que não está no ops não aconteceu (e não é reproduzível).
- **Autonomia e completude (requisito, não meta):** o ops é uma **aplicação/CLI coesa, idempotente e autodescritiva** que instala e opera o sistema inteiro **sem depender da IA**. Um humano — o próprio usuário — provisiona um servidor **do zero** só com o ops. Em projeto grande, `ops install` sobe tudo; `ops update` atualiza tudo; `ops doctor` diagnostica tudo. A IA **usa** o ops; ela não substitui o ops.
- **Superfície mínima de comandos** (nomes ilustrativos, idempotentes, com `--help` e saída machine-readable):
  `bootstrap` (clona os repos no workspace) · `install`/`up` (provisiona e sobe) · `update` (aplica versão do git) · `config` (aplica configuração/segredos via secret manager) · `migrate` (schema, **reversível** com `down`) · `health`/`doctor` (diagnóstico ponta a ponta) · `rollback` · `logs`/`troubleshoot` · `test` (test kit do sistema vivo, §22.1).
- **Correção/hotfix é `ops update`/`ops rollback`** — nunca mão no servidor (§1).
- O ops **sobe com observabilidade integrada** como qualquer ferramenta (§16) e **grava o archive** de cada operação (§28, `<projeto>_archive/`).

## 3. Instalação SEMPRE paralela (= nº de cores)

Instalar/subir um sistema **não pode** levar 20 min em série.

- O ops instala/sobe **módulos e microserviços em paralelo**, com grau de paralelismo = **número de cores da máquina** (`nproc`). Configurável por `--jobs N` / `OPS_JOBS`, **default `nproc`**.
- É o análogo em runtime do fan-out de orquestração (`references/orquestracao.md`): decompõe por **serviço independente** e dispara em ondas de `nproc`, com barra/relatório de progresso.
- **Serialização só onde há dependência real e declarada** (ex.: migração de schema antes do serviço que a consome) — e mesmo aí, o **mínimo** necessário, **explícito** no grafo de instalação, nunca "serializa tudo por via das dúvidas".

## 4. Independência é invariante — falha no paralelo = bug (PRIORIDADE MÁXIMA)

Se a instalação/subida **paralela falha** (race, porta/arquivo/lock disputado, um serviço que não sobe sem outro, ordem implícita não declarada), isso **prova** que os serviços **não são completamente independentes** — e isso fere o piso de **independência de runtime** (`CLAUDE.md` piso 10, `arquitetura.md` §2) e o de **sem monólito distribuído** (piso 6).

- **Corrigir a independência é PRIORIDADE MÁXIMA** — acima de entregar feature. Não é bug de "conveniência de instalação"; é defeito arquitetural que a instalação paralela apenas **revelou**.
- **O ops NÃO contorna serializando "pra funcionar".** Serializar pra mascarar a dependência é **proibido** — esconde o defeito. O ops **expõe** a colisão (relatório: o que disputou o quê, quem esperou quem) e a correção real é feita: **cada serviço sobe sozinho**, sem exigir outro no boot, com **degradação graciosa** (outbox/retry/fila; piso 10). Só depois o paralelo volta a `nproc`.
- Regra: encontrou erro que **só** acontece em paralelo → **para a feature, conserta a independência**. É o trabalho mais urgente que existe no sistema naquele momento.

## 5. Integração com o resto da casa

| Tema | Onde |
|---|---|
| `<projeto>_ops` como control plane (bootstrap/update/manutenção) | `arquitetura.md` §2 |
| Deploy, ambientes, artefato imutável, rollback, preview | `operacao.md` §21 |
| Test kit do sistema vivo (roda pelo ops) | `testes.md` §22.1 |
| Observabilidade integrada do ops | `observabilidade.md` §16 |
| Independência de runtime / sem monólito distribuído | `CLAUDE.md` pisos 10 e 6 |
| Paralelização (o análogo em execução do agente) | `references/orquestracao.md` |
| Archive de cada operação | `operacao.md` §28 (`<projeto>_archive/`) |

Comando: **`/eng-ops`** — audita/scaffolda o ops, verifica o fluxo de ambientes, a paralelização (`nproc`) e a independência.

> Regra de bolso: **nada toca o servidor fora do ops, e nada chega ao servidor sem passar por git + teste local.** Instalar é paralelo por padrão; se o paralelo quebra, o bug é de independência — e é a correção mais urgente que existe.
