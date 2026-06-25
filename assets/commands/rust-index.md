---
description: (Re)gera o índice de microfunções (§39) a partir dos doc-comments
argument-hint: "[dir de origem, ex: internal]"
---

Atualize o **índice de funcionalidades** (§39), que é fonte da verdade do projeto.

1. Rode `node scripts/build-index.mjs ${ARGUMENTS:-src}` e grave a saída em
   `<projeto>_archive/index/INDEX_FUNCTIONS.md`.
2. Se o script sair com código 1, há função sem doc-comment de contexto (§6):
   liste-as e corrija na origem (o quê + onde é usada), não no índice.
3. Revise o `INDEX_GLOBAL.md` (mantido à mão): repos/pastas/o que faz/como se
   comunica continuam corretos? Atualize se a mudança mexeu na estrutura.
4. Confirme que o índice reflete o estado atual — índice desatualizado é bug (§39).
