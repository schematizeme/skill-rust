# Padrões de Código — limites, granularidade, comentários e MAPA

Piso normativo de **organização do código**, válido para toda skill da casa
(backend, frontend, qualquer linguagem). É inegociável: vale para código humano e
gerado por IA. O gate `/<slug>-review` (DoD) reprova o que violar isto.

## 1. Limite de 300 linhas por arquivo

- **Nenhum arquivo de código passa de 300 linhas** (sem contar imports/licença no
  topo). Passou → **quebre em mais de um arquivo**, por coesão (uma
  responsabilidade por arquivo), não por corte arbitrário no meio de uma função.
- **Escopo: código-fonte.** Vale para arquivo de código em qualquer linguagem
  (Rust, Go, TS, Python, shell). **Fora do escopo** (não disparam o gate):
  documentação e Markdown (references, README, ADR, archive, MAPA), config,
  lockfiles/snapshots gerados e fixtures — esses seguem bom senso de tamanho, mas
  o gate de 300 linhas mede **código**, não texto. O CI afere arquivos de código.
- Quebrou um arquivo? Atualize o **MAPA** (§4) no mesmo PR.

## 2. Uma função por arquivo (lógica)

- A regra é **uma função pública/“unidade lógica” por arquivo**. O arquivo existe
  para entregar aquela função; auxiliares privadas pequenas da mesma unidade podem
  conviver, desde que sirvam só a ela e o arquivo siga ≤ 300 linhas.
- **Função > 300 linhas é proibida** — quebre em sub-funções nomeadas, cada uma
  com propósito único, e mova-as para seus próprios arquivos quando crescerem.
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

Todo projeto que segue esta skill mantém um **`MAPA.md`** na raiz (template em
`assets/MAPA.md`) — o guia detalhado de onde está cada coisa. É parte da entrega,
atualizado **no mesmo PR** que mexe no código. Ele lista, para **cada** função — pública e privada, **sem exceção** (uma
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

- [ ] Nenhum arquivo de código > 300 linhas.
- [ ] Uma função/unidade lógica por arquivo; nenhuma função > 300 linhas.
- [ ] Toda função com doc-comment (motivo, comportamento, entradas, saídas, efeitos).
- [ ] `MAPA.md` atualizado no mesmo PR (camada global à mão + microfunções gerada).
- [ ] Índice de microfunções **exaustivo**: uma entrada por função, `nº entradas == nº funções` do serviço (`/<slug>-index` **reprova** se faltar e lista as ausentes); nenhuma órfã.
- [ ] **Grafo** presente: serviços (quem chama quem) + chamadas por função (Mermaid + adjacência).
