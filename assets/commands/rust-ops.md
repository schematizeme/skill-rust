---
description: schematize-rust — audita/scaffolda o <projeto>_ops (interface única) e verifica o fluxo de ambientes, a instalação paralela (nproc) e a independência dos serviços
argument-hint: "[bootstrap | audit | install]"
---

Governe a operação do sistema pelo **`<projeto>_ops`** (`references/ops.md`). Plan-first: **audita, mostra o plano, pede aprovação, então executa** — e **nada toca o servidor fora do ops**.

## 1. Fluxo de ambientes (verifique primeiro)
Confirme que o pipeline força **dev local → teste local → GitHub → hml → prd** e que **nada vai direto pra hml/prd**:
- Deploy só aceita **artefato com proveniência git** (commit SHA); artefato sem origem no git é recusado.
- hml/prd com filesystem **read-only** (sem edição manual); **drift detection** ligado (o ops recusa/alerta divergência com o git).
- **VETADO editar código direto no servidor.** Achou edição manual → trate como incidente (archive), não como fluxo.

## 2. O ops é a interface única (100%)
Verifique/scaffolde a CLI do `<projeto>_ops` com comandos **idempotentes**, `--help` e saída machine-readable, cobrindo **todo** o ciclo sem depender da IA:
`bootstrap` · `install`/`up` · `update` · `config` · `migrate` (reversível) · `health`/`doctor` · `rollback` · `logs`/`troubleshoot` · `test` (test kit §22.1).
- Se alguma operação de servidor **não** tem comando de ops, o gap é o achado: **crie o comando** — não faça por fora (`ssh` ad-hoc, editar arquivo, `docker`/`kubectl` na mão são vetados).
- Meta de completude: **`ops install` provisiona um servidor do zero** sozinho.

## 3. Instalação paralela (= nproc)
A instalação/subida roda **em paralelo**, grau = **`nproc`** (default; `--jobs N`/`OPS_JOBS` sobrepõe). Decomponha por **serviço independente** e dispare em ondas de `nproc`. Serialização só onde há dependência **real e declarada** (ex.: migração antes do consumidor), mínima e explícita.

## 4. Independência (invariante — prioridade máxima)
Rode a instalação **em paralelo de propósito** para provar independência. Se **qualquer** erro só acontece em paralelo (race, porta/lock/arquivo disputado, serviço que não sobe sem outro):
- **PARE a feature.** É bug de **independência de runtime** (piso 10 / §2), não de instalação.
- O ops **expõe** a colisão (relatório: o que disputou o quê, quem esperou quem). **Proibido serializar pra mascarar.**
- Corrija: cada serviço sobe **sozinho**, sem exigir outro no boot, com degradação graciosa (outbox/retry/fila). Só então o paralelo volta a `nproc`.

## 5. Saída
Grave o plano/relatório em `<projeto>_archive/` (§28): estado do fluxo de ambientes, cobertura de comandos do ops (o que falta), tempo de install e grau de paralelismo, e qualquer colisão de independência encontrada (com a correção priorizada). Confirme ao usuário: fluxo ok? ops cobre 100%? install paralela em `nproc`? independência provada?
