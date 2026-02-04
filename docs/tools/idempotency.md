# Idempotência — evitar efeitos colaterais duplicados

Idempotência é crucial em sistemas distribuídos para que reentregas ou duplicações não causem efeitos indesejados.

## Estratégias comuns

- **Chave de idempotência (idempotency key)**: produtor inclui um `idempotency_key` único por operação; consumidor verifica se a chave já foi processada.
- **Upserts / constraints no banco**: use `INSERT ... ON CONFLICT DO NOTHING` ou unique constraints para evitar duplicação.
- **Armazenamento de deduplicação**: mantenha um store (Redis, DB) de IDs processados com TTL para operações recentes.

## Exemplo (Postgres upsert)

```sql
INSERT INTO payments (id, amount, status) VALUES ($1, $2, 'created')
ON CONFLICT (id) DO UPDATE SET amount = EXCLUDED.amount
RETURNING *;
```

## Idempotent producer vs idempotent consumer

- **Producer idempotent** (ex.: Kafka idempotent producer) evita duplicação ao re-tentar envios.
- **Consumer idempotent** garante que reprocessamentos não alterem estado indevidamente.

## Considerações de retenção e armazenamento

- Deduplicação via store exige TTL ou limpeza para evitar crescimento indefinido.
- Para workflows longos, persista estado do processamento (state machine) com checkpoints.

## Boas práticas

- Defina claramente a granularidade da idempotência (por request, por negócio, por usuário).
- Combine idempotency keys com confirmação transacional quando possível.

Referências: artigos sobre idempotency keys, padrões de design de mensageria.
