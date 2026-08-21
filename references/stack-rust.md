# Stack — Rust principal, Go auxiliar

Define a preferência de linguagem desta skill e o ferramental. É a inversão da
a base: aqui **Rust é a linguagem do serviço** e outra do rol pode entrar como **auxiliar** (a
auxiliar**. Tudo o mais (arquitetura, segurança, testes, operação) segue os
references comuns.

## 1. Ordem de preferência

1. **Rust** — padrão para serviços, APIs, workers, CLIs e qualquer backend novo.
2. **Go** — auxiliar. Use quando houver razão concreta e registrada em ADR:
   - SDK/cliente oficial maduro só em Go (sem binding Rust estável);
   - módulo/serviço existente já em Go cujo time é dono;
   - prazo/risco em que o ecossistema Go reduz custo de forma mensurável.
3. **Node backend: proibido.** Frontend Node é 100% permitido (ver `schematize-web`).
4. **PHP: proibido**, migra sumariamente.

A escolha por Go **não** é gosto: precisa de ADR (`assets/ADR.md`) com o motivo.

## 2. Quando trocar de linguagem (medido por funcionalidade)

Mesma régua da casa, aplicada à direção Rust↔Go:

- Ajuste pontual em módulo Go legado **não** porta para Rust.
- ~30% do módulo afetado → extrai a parte mexida para um módulo Rust à parte.
- ~50% já extraído → migra o resto do módulo para Rust.
- Serviço novo: Rust, salvo ADR justificando Go.

## 3. Toolchain Rust (piso)

- **Edição/versão:** Rust estável recente, `edition = "2021"`+; toolchain fixada
  via `rust-toolchain.toml` no repo.
- **Async:** Tokio. **HTTP:** axum (ou similar com a mesma fronteira clara).
- **Banco:** `sqlx` (queries parametrizadas, checagem em tempo de compilação) ou
  `diesel`; **nunca** SQL por `format!`/concatenção.
- **Erros:** `Result`/`?` sempre; erro de domínio tipado (`thiserror`), erro de
  borda agregável (`anyhow`) só nas pontas. **Proibido** `.unwrap()`/`.expect()`
  em caminho de produção para silenciar; sem `let _ =` engolindo `Result`.
- **Lints travando CI:** `cargo clippy -- -D warnings` e `cargo fmt --check`.
  `#![deny(unsafe_code)]` por padrão; `unsafe` só com ADR e bloco comentado
  explicando invariantes.
- **Testes:** `cargo test` + (no domínio crítico) property-based (`proptest`) e
  mutation; cobertura é piso, não meta. Detalhe em `references/testes.md` (o recorte Rust) — a **disciplina** de teste é da `schematize-qa`, e a segurança ofensiva da `schematize-pentest`.
- **Segurança de deps:** `cargo deny`/`cargo audit` no CI; dependência nova com
  nome/licença/versão verificados (typosquatting é real).

## 4. Outra linguagem do rol como auxiliar (piso quando usada)

Um serviço auxiliar em outra linguagem do rol sancionado segue **integralmente a skill irmã dela**
— e os pisos são os mesmos em todas: erro nunca engolido, SQL parametrizado, container não-root,
lint da linguagem no CI, IAM como app separada, efeito externo que não sai de não-produção.
A escolha do auxiliar é **fit + ADR**, como a do principal
(`schematize-engineering` → `references/linguagens.md`), não preferência.

## 5. Pisos de código valem igual

Independente da linguagem, valem os limites de `references/padroes-codigo.md`:
arquivos ≤ 750 linhas (~500 de código útil + ~250 de comentário; flag em > 300
úteis), uma função/unidade lógica por arquivo, **todo** item com doc-comment
(`///` em Rust) explicando motivo,
comportamento esperado, entradas, saídas e efeitos, e o **`MAPA.md`** da
aplicação atualizado no mesmo PR.

## 6. Coexistência com as outras skills

As skills do rol e a `schematize-web` podem estar habilitadas na mesma máquina ao
mesmo tempo. Não há conflito: cada skill instala em seu próprio diretório
(`.claude/skills/schematize-<slug>/`) e seus comandos são prefixados pelo slug
(`/rust-*`, `/elixir-*`, `/cs-*`, `/web-*`, …). Escolha a skill pela **linguagem do
serviço** que você está tocando — não por preferência; a linguagem já foi decidida
por fit + ADR na `schematize-engineering` (`references/linguagens.md`).
