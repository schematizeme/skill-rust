# Eventos, Banco de Dados, Cache, APIs, Resiliência e Jobs

> Parte da skill **schematize-rust**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 9. Eventos e Mensageria
- 10. Banco de Dados
- 11. Cache
- 12. APIs
- 18. Resiliência
- 19. Jobs e Workers

---

## 9. Eventos e Mensageria

**MUST — produção e consumo**
- Eventos são imutáveis e versionados (`v1`, `v2`).
- Consumidores idempotentes (deduplicação por id de evento).
- Suporte a replay.

**MUST — compatibilidade evolutiva**
- Novos campos são **opcionais**, com default seguro.
- Nunca remover ou renomear campo sem **nova versão** do evento.
- Consumidores **ignoram campos desconhecidos** (forward compatible).
- Coexistência de versões durante janela de migração documentada.

**MUST — entrega**
- **Transactional Outbox** para publicação de eventos. Dual-write (banco + broker no mesmo fluxo) é **VETADO** (ver §37).
- DLQ (dead letter queue) configurada para todo consumidor.
- **Retry infinito proibido.** Política de retry explícita com limite e backoff.

**MUST — backpressure**
- Consumidores com limites explícitos de concorrência e prefetch.
- Throttling em produtores quando broker sinaliza pressão.

**Convenção de nome:** `<dominio>.<entidade>.<evento>` no passado.
Exemplos: `catalog.product.created`, `billing.invoice.paid`.

**Brokers**

| Cenário | Stack |
|---|---|
| Eventos leves, baixa latência, pub/sub simples | **NATS** |
| Alto throughput, retenção, replay, streaming | **Kafka** |
| Workflows com routing complexo (caso a caso) | RabbitMQ |

> NATS e Kafka são padrão. RabbitMQ exige justificativa pelo custo operacional.

---

---

## 10. Banco de Dados

| Caso | Stack |
|---|---|
| Relacional | PostgreSQL |
| Cache | Redis |
| Busca textual | OpenSearch |
| Analytics / eventos | ClickHouse |

**MUST**
- Migrations versionadas, **reversíveis** (com `down`), automatizadas no deploy.
- Schema próprio por serviço.
- Sem joins cross-service no banco.
- **Timestamps em UTC, sempre.** Conversão de timezone apenas na borda (UI/API response).
- **IDs:** UUIDv7 ou ULID por padrão. IDs sequenciais apenas com justificativa (ex: ordenação natural exigida pelo domínio).
- **Toda query com input externo é parametrizada.** Concatenação de string em SQL é **VETADA** (§37).

**SHOULD — alta escala**
- Separação leitura/escrita (réplicas) quando volume justificar.
- Revisão periódica de índices.
- Análise de query plan em endpoints críticos.
- Particionamento para tabelas com crescimento previsível alto.

**Ferramentas sugeridas:** `sqlx-cli`/`refinery` (Rust), `golang-migrate`, `node-pg-migrate`, `flyway`.

---

---

## 11. Cache

**MUST**
- TTL **sempre explícito**. Sem TTL infinito sem ADR.
- Mitigação de cache stampede (single-flight, lock, jitter).
- Fallback seguro em cache miss — degradação graciosa, nunca falha total.
- Invalidação documentada por chave.
- Resposta autenticada **sempre** tem a chave de cache segmentada por usuário e tenant (§37).

**MUST NOT**
- Cache como source of truth.
- Cache de dados pessoais sensíveis sem criptografia.

---

---

## 12. APIs

**MUST**
- **OpenAPI 3.1** em `/docs/openapi.yaml` — fonte da verdade.
- Versionamento na URL: `/v1`, `/v2`.
- Validação de payload na borda.
- Erro padrão (compatível com RFC 7807):

```json
{
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product not found",
    "trace_id": "01HXYZ...",
    "details": []
  }
}
```

- Operações de escrita aceitam `Idempotency-Key` **e o implementam de fato** (aceitar o header e ignorar é VETADO — §37).
- Quebra de contrato exige nova versão + janela de deprecação ≥ 90 dias com headers `Deprecation` e `Sunset`.
- Timestamps em **ISO-8601 com timezone explícito** (preferencialmente `Z`).

**MUST — paginação**
- **Cursor pagination** é o padrão.
- Offset apenas em listas pequenas (< 10k itens) e estáticas.
- Resposta inclui `next_cursor` e `has_more`.

**MUST — rate limiting**
- Rate limiting **distribuído** (Redis, gateway, ou serviço dedicado).
- Chave por usuário, API key e tenant.
- Resposta 429 com header `Retry-After`.

**SHOULD**
- Contratos consumer-driven (Pact) entre serviços.
- gRPC para alta performance interna; HTTP/JSON para externa.

---

---

## 18. Resiliência

**MUST em chamadas externas:**
- Timeout explícito (nunca infinito).
- Retry com backoff exponencial + jitter, idempotência garantida.
- Circuit breaker.
- Rate limiting na borda.
- Bulkhead em integrações críticas.

**MUST — independência e falha de dependência (resiliência por design):**
- **Serviço sobe e opera sozinho.** A ausência/queda de outra dependência (serviço, broker, cache) **nunca** impede o boot nem derruba este serviço — degrada (fallback, resposta parcial, enfileira e segue), não crasha em cascata (§2). "O `ledger` não sobe sem o `core`" é bug de acoplamento, não requisito.
- **Falha ao chamar/notificar outro serviço não se perde nem trava a cadeia.** Ao não conseguir notificar B, o serviço A **obrigatoriamente**: (1) **persiste o intento/dado em store durável** para retomar — outbox no mesmo commit, ou fila/Redis/tabela de pendências; nunca só em memória; (2) **loga com `trace_id`**; (3) **dispara alerta** (Grafana/alertmanager) pro ops/dev; (4) **retoma** com retry+backoff+jitter idempotente até o limite; estourou → **DLQ** + escala pro humano. Nunca falha em silêncio, nunca perde o dado, nunca deixa a cadeia parada sem sinal.

---

---

## 19. Jobs e Workers

**MUST**
- Jobs **idempotentes** (pode executar 2x sem corromper).
- Timeout obrigatório por job.
- Política de retry explícita com limite.
- Progress/state persistido para jobs longos ou críticos.
- DLQ para jobs que falharem após max retries.

**SHOULD**
- Jobs longos divisíveis em chunks com checkpoint.
- Cancelamento gracioso (respeitar context/signal).

---

---
