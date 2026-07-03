# MAPA da aplicação — <project>

Guia detalhado de **onde está cada coisa, para que serve e como entra/sai**.
Obrigatório por `references/padroes-codigo.md` (§4). Atualize no mesmo PR que
mexe no código. Duas camadas:

- **Global** — mantido à mão (repos, pastas, módulos, comunicação, entradas/saídas).
- **Microfunções** — gerado por `/<slug>-index` a partir dos doc-comments.

---

## 1. Visão geral

- **O que é:** <uma linha sobre o sistema>
- **Stack:** <linguagem(s), framework, banco, fila>
- **Pontos de entrada:** <rotas HTTP / CLIs / jobs / consumers>
- **Pontos de saída:** <banco / filas / APIs externas / arquivos>

## 2. Mapa de módulos (global, à mão)

| Módulo / pasta | Responsabilidade | Depende de | É usado por |
|---|---|---|---|
| `caminho/` | <o que faz> | <módulos> | <chamadores> |

## 3. Fluxos principais (entrada → saída)

Para cada fluxo de negócio, a cadeia ponta a ponta:

- **<fluxo>**: entrada `<rota/evento>` → `<função/arquivo>` → `<função/arquivo>`
  → saída `<banco/fila/resposta>`

## 4. Microfunções (GERADO — não editar à mão)

> Preenchido por `/<slug>-index` a partir dos doc-comments (§3 de
> `padroes-codigo.md`). Cada entrada traz: onde está, para que serve,
> dependências, auxiliares/chamadores, entrada e saída.

<!-- BEGIN: index-funcoes -->
<!-- (gerado) -->
<!-- END: index-funcoes -->
