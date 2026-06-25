---
description: schematize-rust — lista todos os comandos disponíveis e o que cada um faz
---

Mostre ao usuário a lista de comandos do conjunto **schematize-rust**, em formato
de tabela legível, exatamente com este conteúdo (ajuste se houver comandos novos
instalados em `.claude/commands/`):

| Comando | O que faz |
|---|---|
| `/rust-help` | Lista todos os comandos do schematize-rust (este). |
| `/rust-cc` | Context compact: gera `context.md` + `checklist.md` em `<projeto>_archive/context/` e roda `/compact`. |
| `/rust-handoff` | Gera o handoff (`context.md` + `checklist.md`) **sem** compactar — ideal pra fim de sessão ou troca de tarefa. |
| `/rust-qa` | Fluxo de Q.A. plan-first (§22.9): planeja tudo, gera MD de passo a passo, pede aprovação, e roda faseado/assistido ou de uma vez. |
| `/rust-review` | Roda o gate da Definition of Done e dos anti-padrões (§35, §37): arquivos >300 linhas, função sem doc-comment, índice desatualizado, macaquices de segurança. |
| `/rust-index` | (Re)gera o índice de microfunções (§39) a partir dos doc-comments das funções. |

Depois da tabela, diga em uma linha que o detalhe normativo está na skill
`schematize-rust` (referências em `references/`) e que o site é `skills.schematize.me/rust`.
