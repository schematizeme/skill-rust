# Cadeia de Suprimentos de Software (supply chain)

> Piso de **segurança da cadeia de suprimentos** — consolida num só lugar o que
> antes ficava espalhado (higiene de dependência em `seguranca.md`, imagem/deploy
> em `operacao.md`). O ataque hoje raramente é no seu código: é numa dependência,
> numa imagem base ou no pipeline. Trate build e dependências como superfície de
> ataque de primeira classe.

## Índice
- S1. Dependências
- S2. SBOM e vulnerabilidades
- S3. Imagem de container
- S4. Pipeline e proveniência (SLSA)
- S5. Segredos no build

---

## S1. Dependências

**MUST**
- **Lockfile commitado** (`Cargo.lock` para binário, `go.sum`, `package-lock`),
  build **offline/reproduzível** a partir dele. Sem range frouxo em dependência
  sensível.
- **Verificar nome e licença** de toda dependência nova: typosquatting é real
  (nome quase igual ao popular); licença na allowlist (MIT/Apache-2.0/BSD/MPL-2.0/
  ISC ok; GPL/AGPL/SSPL/proprietária só com ADR).
- **Minimizar superfície:** menos dependência transitiva é menos risco. Avalie peso
  e manutenção antes de adicionar; corte o que não usa.

**SHOULD**
- Atualização automatizada (Dependabot/Renovate) com CI verde como gate.
- `cargo vendor`/proxy de módulos para builds críticos não dependerem de upstream no ar.

**VETADO**
- Instalar de fonte não confiável / `curl | sh` de origem não verificada no build.
- Fixar dependência por branch (`main`) em produção.

---

## S2. SBOM e vulnerabilidades

**MUST**
- **Gerar SBOM** (CycloneDX/SPDX) no build e **versioná-lo junto ao artefato** —
  é o inventário do que foi de fato embarcado. `cargo cyclonedx` / `syft` servem.
- **Scan de vulnerabilidade no CI, travando o merge** em `high`/`critical` sem ADR
  de aceite: `cargo audit`/`cargo deny` (Rust), `govulncheck` (Go), `npm audit`/SCA
  (Node). Scan também na imagem (`grype`/`trivy`).
- Vulnerabilidade aceita conscientemente vira ADR com prazo de correção, não um
  silêncio.

**SHOULD**
- `cargo deny` também para licença e fontes (advisories + bans num só gate).

---

## S3. Imagem de container

**MUST**
- **Base mínima e pinada por digest** (`@sha256:...`), não por tag móvel (`latest`/
  `1`). Distroless/Alpine/scratch quando viável; menos pacote, menos CVE.
- **Não-root, filesystem read-only**, sem capabilities desnecessárias (liga com o
  piso de container do `operacao`/segurança).
- **Multi-stage build:** toolchain só no estágio de build; a imagem final carrega só
  o binário e o necessário em runtime.

**SHOULD**
- Rebuild periódico pra absorver patch da base; recscan da imagem publicada.

---

## S4. Pipeline e proveniência (SLSA)

**MUST**
- **Build só no CI** (não na máquina do dev pra produção), a partir do código
  versionado; o pipeline é parte da TCB (base de confiança).
- **Assinar o artefato/imagem** (cosign/sigstore) e **verificar a assinatura na
  admissão** (deploy só aceita imagem assinada por pipeline confiável). Assinar sem
  verificar não protege.
- **Proveniência:** emitir atestação de build (quem buildou, de qual commit, com
  qual SBOM) — alvo **SLSA** crescente. Tag de imagem inclui o commit.

**SHOULD**
- Permissões mínimas no CI (token escopado, sem segredo amplo); branch protegida e
  review obrigatório antes de buildar release.

---

## S5. Segredos no build

**MUST**
- **Nenhum segredo no artefato/imagem** nem em layer intermediária (layer é
  inspecionável). Segredo de build via mecanismo efêmero (BuildKit secret mount),
  nunca `ARG`/`ENV` persistido nem `COPY .env`.
- **Secret scan no pipeline** (gitleaks/trufflehog) travando vazamento antes do
  merge. `.env` real fora do git.

> Regra de bolso: **você entrega tudo que embarca.** Lockfile + SBOM + scan que
> trava + imagem mínima/pinada/assinada + build no CI sem segredo no layer. Se não
> dá pra dizer exatamente o que tem dentro do artefato e quem o produziu, a cadeia
> está aberta.
