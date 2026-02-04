# Kafka — Conceitos e práticas para streaming

Apache Kafka é uma plataforma distribuída de streaming orientada a tópicos, projetada para alto débito e retenção de mensagens.

## Conceitos principais

- **Topic**: sequência de mensagens categorizadas por nome.
- **Partition**: divisão do tópico que permite paralelismo; cada mensagem tem um offset.
- **Producer**: publica mensagens no tópico.
- **Consumer**: lê mensagens de tópicos; consumidores pertencem a **consumer groups**.
- **Broker**: nó que armazena partições; múltiplos brokers formam um cluster.

## Consumo e offsets

- Consumers em um mesmo group repartem partições — cada partição é consumida por um único consumer no group.
- Offset é posição no log; commit de offset indica progresso.
- Estratégias: `auto-commit` (conveniência) vs commit manual (mais controle para at-least-once).

## Retenção e compactação

- Configure `retention.ms` para tempo de retenção ou `retention.bytes` por tamanho.
- `log.compaction` mantém apenas a última mensagem por chave (útil para changelogs/state).

## Replicação e tolerância a falhas

- Partições têm réplicas; `min.insync.replicas` e `acks` do producer influenciam durabilidade.

## Performance e tuning

- Partições aumentam paralelismo; balanceie número de partições com consumidores.
- Ajuste `batch.size`, `linger.ms` no producer para throughput vs latência.

## Exactly-once / Idempotência

- Kafka suporta `idempotent producer` e `transactions` para exatamente-once semantics entre topics (mais no guia de idempotência).

## Operação e monitoramento

- Monitore lag de consumer (offset difference) para detectar atrasos.
- Use ferramentas (kafka-manager, Confluent Control Center) e métricas Prometheus.

## Exemplo (Python, confluent-kafka)

```python
from confluent_kafka import Producer

conf = {'bootstrap.servers':'kafka:9092', 'client.id':'meu-produtor'}
p = Producer(conf)
p.produce('events', key='user-1', value='{"action":"created"}')
p.flush()
```

Referências: documentação Apache Kafka e Confluent.
