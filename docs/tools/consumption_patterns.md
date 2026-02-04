# Padrões de Consumo — padrões comuns em mensageria

Listagem de padrões de consumo e quando usá-los.

## Competing Consumers (Work Queue)

- Vários consumers leem da mesma fila; cada mensagem é processada por apenas um consumer.
- Uso: processamento paralelo de tarefas (jobs/background).

## Publish/Subscribe

- Mensagens são publicadas e entregues a múltiplos subscribers (fanout ou topics).
- Uso: notificações, eventos de sistema.

## Routing

- Use exchanges `direct`/`topic` (RabbitMQ) ou chaves (Kafka keys) para rotear mensagens a consumidores específicos.

## Request/Reply (RPC)

- Padrão onde o produtor espera uma resposta: use uma fila de reply com correlação (`correlation_id`) e timeout.

## Dead-letter / Retry

- Implementar filas de DLX para mensagens que falham repetidamente.
- Estratégias de retry: immediate, delayed (via TTL), exponential backoff.

## Idempotency & Exactly-once patterns

- Detectar duplicatas via `message_id` / chave de idempotência.
- Use transações (Kafka) ou mecanismos de dedup/unique constraint em consumidores.

## At-least-once vs At-most-once vs Exactly-once

- At-least-once: garante entrega, pode ocorrer duplicação (use idempotência).
- At-most-once: entrega no máximo uma vez — risco de perda (menos reprocessos).
- Exactly-once: mais complexo; Kafka transactions ou coordenação externa podem ser usados.

## Observabilidade

- Monitore lag, taxa de erros, retries e tamanho de filas.
