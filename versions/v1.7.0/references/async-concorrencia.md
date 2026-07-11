# Async e Concorrência (Rust)

> Parte da skill **schematize-rust**. Os erros mais caros de um backend Rust não
> são de compilação — são de runtime async: bloquear o executor, cancelamento que
> corrompe estado, deadlock de lock, e backpressure ausente. Esta reference é o
> piso de concorrência. Liga com `stack-rust.md` (Tokio/axum) e `dados-eventos.md`
> (resiliência, jobs).

## Índice
- A1. Não bloquear o runtime
- A2. Cancelamento e `select!`
- A3. Locks, `Send`/`Sync` e estado compartilhado
- A4. Backpressure e canais
- A5. Tarefas, erros e shutdown

---

## A1. Não bloquear o runtime async

**VETADO**
- **Trabalho bloqueante numa task async** sem isolar: I/O de arquivo síncrono, CPU
  pesado, `std::thread::sleep`, chamada de lib bloqueante (driver sync, regex
  gigante, criptografia pesada) dentro de `async fn`. Isso **trava uma thread do
  executor** e mata a latência de todas as outras tasks.
- **`block_on` dentro de contexto async** (reentrância no runtime) → pânico/deadlock.

**MUST**
- CPU pesado / lib bloqueante → `tokio::task::spawn_blocking` (ou um pool dedicado).
- Espera → `tokio::time::sleep`, nunca `std::thread::sleep`.
- Operações longas têm **timeout** (`tokio::time::timeout`) — nada espera pra sempre.

**SHOULD**
- Medir: uma task que segura a thread por > ~100µs sem `.await` é candidata a
  `spawn_blocking` ou a ser quebrada (liga com §6, função pequena).

---

## A2. Cancelamento e cancel-safety

Em Rust async, uma future é **cancelada** ao ser dropada (timeout, `select!` que
escolheu outro ramo, cliente que desconectou). Cancelar no meio é normal — o
código precisa ser **seguro a cancelamento**.

**MUST**
- **Não deixar invariante quebrada se a future for dropada no meio.** Estado
  parcial (incremento feito mas não confirmado, lock adquirido, arquivo
  semi-escrito) é bug. Use commit atômico / transação / `Drop` que limpa.
- **`tokio::select!` só com ramos cancel-safe.** Operação não-cancel-safe (ex.: ler
  de um stream que perde bytes ao dropar) não vai crua num `select!` — encapsule
  num task com canal, ou use a variante cancel-safe documentada.
- **Propagar `CancellationToken`** (ou o shutdown signal) pra encerrar trabalho em
  andamento de forma limpa, em vez de abortar no escuro.

**VETADO**
- Assumir que o corpo de um ramo de `select!` "termina" — o ramo perdedor é dropado
  exatamente onde estava o `.await`.

---

## A3. Locks, `Send`/`Sync` e estado compartilhado

**MUST**
- **Nunca segurar um lock através de um `.await`.** Manter um `std::sync::Mutex`
  (ou mesmo `tokio::sync::Mutex`) travado enquanto aguarda é deadlock/contention
  esperando acontecer. Padrão: pegue o lock, copie/mute o necessário, **solte
  antes** do `await`.
- **Escolha o lock certo:** `std::sync::Mutex` pra seção crítica curta e sem await
  (mais rápido); `tokio::sync::Mutex` só quando precisa segurar **através** de await
  (raro — prefira reestruturar); `RwLock` quando há muito mais leitura que escrita.
- **Estado compartilhado entre tasks = `Arc<...>`**; mutável = `Arc<Mutex<...>>` /
  `Arc<RwLock<...>>`. Tipos cruzando `tokio::spawn` precisam ser `Send + 'static`.

**SHOULD**
- Preferir **passar mensagem a compartilhar memória** (canal + uma task dona do
  estado) quando a contenção de lock aparecer — geralmente é mais simples e rápido.
- Ordem de aquisição de locks consistente (evita deadlock A→B / B→A).

**VETADO**
- `Arc<Mutex<>>` global virando ponto único de contenção sem necessidade.

---

## A4. Backpressure e canais

**MUST**
- **Canais e filas são limitados (`bounded`).** `mpsc::channel(n)` com `n` definido;
  produtor mais rápido que consumidor **aguarda** (backpressure), não acumula
  memória sem limite. `unbounded_channel` é VETADO em caminho de produção sem
  justificativa em ADR (é OOM esperando carga).
- **Política de cheio explícita:** aguardar, descartar (com métrica), ou rejeitar —
  decidida, não acidental.

**SHOULD**
- Concorrência limitada em fan-out: `Semaphore` ou `buffer_unordered(n)` (streams)
  pra não disparar 10k requisições de uma vez contra um upstream.

---

## A5. Tarefas, erros e shutdown

**MUST**
- **`JoinHandle` é checado.** `tokio::spawn` cujo resultado/pânico é ignorado
  esconde falha — agregue (`JoinSet`) e trate o erro/pânico de cada task.
- **Erro nunca engolido** (§ pisos): task que falha reporta (log + métrica) e
  decide retry/parar — não morre em silêncio.
- **Graceful shutdown:** ao receber SIGTERM, parar de aceitar novo trabalho, sinalizar
  cancelamento, **drenar** o que está em voo com timeout, e só então sair. Conexões
  e flush de buffer fechados ordenadamente (liga com `references/operacao.md`).

**SHOULD**
- `tracing` com spans por task/requisição (instrumentar `async fn`) pra observar
  concorrência real (liga com observabilidade).

> Regra de bolso: **não bloqueie o executor, não segure lock no `await`, limite
> toda fila, e assuma que qualquer future pode ser cancelada no pior ponto.** Em
> Rust o compilador garante memória; concorrência correta é com você.
