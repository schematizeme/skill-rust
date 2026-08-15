# Padrões de Código — limites, granularidade, comentários e MAPA

Piso normativo de **organização do código**, válido para toda skill da casa
(backend, frontend, qualquer linguagem). É inegociável: vale para código humano e
gerado por IA. O gate `/<slug>-review` (DoD) reprova o que violar isto.

## 1. Tamanho de arquivo — teto de 750 linhas (≤ 500 de código útil + ~250 de comentário)

Regra **em camadas**: um teto duro generoso e um **flag** mais cedo que sinaliza
cheiro de função extensa. A ideia é dar espaço pra código bem documentado sem
soltar a mão da granularidade.

- **Teto DURO: 750 linhas por arquivo de código** (sem contar imports/licença no
  topo). Desse orçamento, **~250 linhas são reservadas a comentário/doc-comment**
  (§3) e **até ~500 são código útil**. Passou de **750 no total** — ou de **~500 de
  código útil** — → **quebre em mais de um arquivo**, por coesão (uma
  responsabilidade por arquivo), nunca por corte arbitrário no meio de uma função.
  O gate `check-diff` (§6) **BLOQUEIA** acima do teto.
- **FLAG em > 300 linhas de código útil** (não bloqueia, mas **sempre sinaliza**):
  se o **código útil** de um arquivo — descontando comentário e linha em branco —
  passar de **300 linhas**, isso é **indício** de que a função/arquivo está **muito
  extensa** e provavelmente **pode ser melhorada ou precisa de um nível de abstração
  maior**. Não trava a entrega; o gate **flagueia sempre** e o ponto entra como
  **dívida técnica pra rever quando as prioridades forem resolvidas** (registre no
  archive/ADR). **Nunca engula o flag** — sinalizar é obrigatório, ignorar não é.
- **Observabilidade tem folga natural (~400 úteis):** arquivos dominados por
  instrumentação (tracing, métricas, logging estruturado) incham por natureza —
  olhando de fora, ~400 linhas úteis é esperado. Pra esses, o flag só é significativo
  acima de **~400** de código útil (ainda assim registrado). O teto duro de 750/500
  continua valendo igual.
- **Escopo: código-fonte.** Vale para arquivo de código em qualquer linguagem
  (Rust, Go, TS, Python, shell). **Fora do escopo** (não disparam o gate):
  documentação e Markdown (references, README, ADR, archive, MAPA), config,
  lockfiles/snapshots gerados e fixtures — esses seguem bom senso de tamanho, mas
  o gate mede **código**, não texto. O CI afere arquivos de código.
- Quebrou um arquivo? Atualize o **MAPA** (§4) no mesmo PR.

## 2. Uma função por arquivo (lógica)

- A regra é **uma função pública/“unidade lógica” por arquivo**. O arquivo existe
  para entregar aquela função; auxiliares privadas pequenas da mesma unidade podem
  conviver, desde que sirvam só a ela e o arquivo siga o teto (§1: ≤ 750 linhas,
  ≤ ~500 de código útil).
- **Função com > 300 linhas de código útil dispara o flag** (§1) — indício de
  extensão excessiva / falta de abstração: quebre em sub-funções nomeadas com
  propósito único e mova-as para seus arquivos quando puder. Se não for a prioridade
  agora, **flagueia e revisita depois**; o teto duro do arquivo continua 750/500.
- Sem “arquivo balaio” (`utils.ts`, `helpers.go`, `commons`) que acumula funções
  sem relação. Nome do arquivo = o que ele faz.

## 3. Tudo comentado: motivo + comportamento esperado

Toda função carrega um **doc-comment** (no formato nativo: `///` em Rust, doc
comment em Go, JSDoc/TSDoc em TS, docstring em Python) que responde, no mínimo:

- **Por quê existe** — o problema que ela resolve / a decisão que encapsula.
- **Como se espera que funcione** — o passo-a-passo em uma frase, pré-condições e
  invariantes.
- **Entradas** — cada parâmetro: o que é, faixa/validação esperada.
- **Saídas** — retorno e seu significado; erros possíveis e quando ocorrem.
- **Efeitos colaterais** — I/O, rede, banco, estado global, mutação.
- **Fluxo do dado (começo → meio → fim)** — **de onde vem** (qual origem/serviço/
  fila/tabela; se vem de **outra aplicação**, nomeie qual e o contrato/evento),
  **o que é feito** (a transformação/decisão esperada), e **pra onde vai** (destino:
  qual serviço/tópico/tabela/resposta). Toda função que **cruza fronteira de
  aplicação** (chama, recebe de, ou notifica outro serviço) diz isso explicitamente
  — ninguém, humano ou agente, deve precisar caçar no código pra descobrir a origem
  e o destino do dado.

Comentário que só repete a assinatura não conta. O doc-comment é o contrato:
quem chama deve entender a função **sem ler o corpo**. O índice de microfunções
(`/<slug>-index`) é **gerado** desses doc-comments e falha o CI se achar função
sem contexto. **Toda funcionalidade é mapeada** (§4): nenhuma fica fora do índice/
MAPA — busca burra que torra token e tempo é sintoma de mapa incompleto, não de
falta de busca.

## 4. MAPA da aplicação (arquivo-guia obrigatório)

Todo projeto que segue esta skill mantém um **`MAPA.md`** em **`<projeto>_archive/index/MAPA.md`** (template em
`assets/MAPA.md`) — **nunca no root do projeto**. Todo MD **gerado** (MAPA, índices, planos, relatórios, handoffs) mora no archive; o root fica limpo (só código, config e os poucos MDs de projeto mantidos à mão: README, `CLAUDE.md`, LICENSE). Layout canônico do archive em `references/operacao.md` (§28). É parte da entrega,
atualizado **no mesmo PR** que mexe no código (o archive é versionado). Ele lista, para **cada** função — pública e privada, **sem exceção** (uma
entrada por função):

- **Onde está** — caminho do arquivo (e símbolo).
- **Para que serve** — propósito em uma linha.
- **Dependências** — o que ela chama (funções/módulos/serviços externos).
- **Auxiliares** — quem a apoia / quem depende dela (chamadores).
- **Entrada e saída** — de onde vêm os dados e para onde vão (args/retorno, rota,
  fila, arquivo, banco).

O MAPA tem duas camadas, espelhando o índice de funcionalidades:

- **Global** (mantido à mão): repositórios, pastas, módulos, como se comunicam,
  pontos de entrada (rotas/jobs/CLIs) e de saída (banco/fila/API externa).
- **Microfunções** (gerado por `/<slug>-index` a partir dos doc-comments §3).

O índice de microfunções é **exaustivo e conferível por contagem**: **uma entrada por função** de **cada** serviço/repo (pública e privada) — o número de entradas **tem que bater** com o número de funções do código (`nº entradas == nº funções`). Menos que isso é **falha**, não "resumo": um MAPA de 90 linhas para 100+ funções está errado. E o MAPA é um **grafo**, não uma lista — traz o **grafo de serviços** (quem chama/notifica quem, por contrato) e o **grafo de chamadas** por função (quem chama / é chamada), como **Mermaid + adjacência**, pra percorrer do ponto de entrada à saída e medir o raio de impacto. Contrato e gate: `references/entrega.md` §39.

Sem MAPA atualizado, o PR não passa na DoD. O objetivo: qualquer pessoa (ou IA)
abre o MAPA e sabe **onde tocar, o que aquilo afeta e por onde entra/sai** antes
de ler o código.

## Checklist (entra na Definition of Done)

- [ ] Nenhum arquivo de código > 750 linhas (nem > ~500 de código útil) — teto duro.
- [ ] Arquivo/função com > 300 linhas de código útil (~400 em observabilidade) **flagueado** e registrado como dívida pra rever (não bloqueia, mas nunca silenciado).
- [ ] Uma função/unidade lógica por arquivo.
- [ ] Toda função com doc-comment (motivo, comportamento, entradas, saídas, efeitos).
- [ ] `MAPA.md` atualizado no mesmo PR, em **`<projeto>_archive/index/`** (nunca no root) — camada global à mão + microfunções gerada.
- [ ] Índice de microfunções **exaustivo**: uma entrada por função, `nº entradas == nº funções` do serviço (`/<slug>-index` **reprova** se faltar e lista as ausentes); nenhuma órfã.
- [ ] **Grafo** presente: serviços (quem chama quem) + chamadas por função (Mermaid + adjacência).
