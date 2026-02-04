# RabbitMQ — Conceitos e práticas

RabbitMQ é um broker AMQP popular para filas, roteamento e patterns de mensageria.

## Conceitos chave

- **Producer**: envia mensagens para um **exchange**.
- **Exchange**: roteia mensagens para filas (`direct`, `fanout`, `topic`, `headers`).
- **Queue**: armazena mensagens até que um consumidor as consuma.
- **Binding**: ligação entre exchange e queue com uma chave de roteamento.
- **Ack/Nack**: consumidor confirma processamento; sem ack a mensagem pode ser reentregue.

## Exchanges comuns

- `direct`: roteamento por chave exata.
- `fanout`: entrega para todas as filas ligadas (pub/sub simples).
- `topic`: roteamento por padrão (ex.: `orders.*`).
- `headers`: roteia por cabeçalhos, menos usado.

## Durabilidade e persistência

- Declare filas como `durable` e publique mensagens com `delivery_mode=2` para persistência em disco.
- Habilite `ha` policies (ou mirrored queues) em clusters para tolerância a falhas (em versões mais novas use quorum queues).

## Confirmação e prefetch

- Use `manual ack` (não `auto_ack`) para garantir que tarefas só sejam removidas após processamento bem-sucedido.
- Ajuste `prefetch_count` (QoS) para limitar número de mensagens simultâneas por consumidor.

## Dead-lettering e retries

- Configure `x-dead-letter-exchange` nas filas para capturar mensagens rejeitadas ou expiradas.
- Implemente backoff exponencial via TTL e filas de retry.

## Segurança e operação

- Use TLS, autenticação forte e usuários com permissões mínimas.
- Monitore filas, publishers e utilizadores com métricas (Prometheus exporter para RabbitMQ).

## Padrões de uso

- Work queue (competing consumers): um exchange direto e múltiplos consumers para processamento paralelo.
- Pub/Sub: `fanout` exchange para broadcast.
- Routing: `topic` para assinaturas por padrão.

## Exemplo (Python, pika)

```python
import pika

conn = pika.BlockingConnection(pika.ConnectionParameters('rabbit'))
ch = conn.channel()
ch.exchange_declare('events', exchange_type='topic', durable=True)
ch.basic_publish('events', routing_key='orders.created', body='{}', properties=pika.BasicProperties(delivery_mode=2))
conn.close()
```

Referências: documentação RabbitMQ e AMQP.
