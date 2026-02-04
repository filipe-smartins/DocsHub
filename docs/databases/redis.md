# Redis — Guia completo e aprofundado

Este documento cobre arquitetura, instalação, persistência, replicação, clustering, tuning, segurança, operações e exemplos práticos do Redis.

## Índice

- [Introdução e casos de uso](#introducao-e-casos-de-uso)
- [Instalação e deployment](#instalacao-e-deployment)
- [Tipos de dados e comandos essenciais](#tipos-de-dados-e-comandos-essenciais)
- [Persistência: RDB vs AOF](#persistencia-rdb-vs-aof)
- [Replicação, Sentinel e Cluster](#replicacao-sentinel-e-cluster)
- [Configuração e tuning de memória / eviction](#configuracao-e-tuning-de-memoria-eviction)
- [Segurança e autenticação](#seguranca-e-autenticacao)
- [Monitoração e operações](#monitoracao-e-operacoes)
- [Scripting (Lua) e Transactions](#scripting-lua-e-transactions)
- [Streams e patterns de consumo](#streams-e-patterns-de-consumo)
- [Módulos e ecossistema](#modulos-e-ecossistema)
- [Boas práticas, pitfalls e scaling](#boas-praticas-pitfalls-e-scaling)
- [Exemplos rápidos (Python)](#exemplos-rapidos-python)
- [Referências](#referencias)

---

## <a id="introducao-e-casos-de-uso">Introdução e casos de uso</a>

Redis é um datastore em memória, rápido, com suporte a estruturas ricas (strings, hashes, lists, sets, zsets, streams, bitmaps, hyperloglog). Muito usado para caching, filas, pub/sub, rate limiting, sessões e como primary datastore em use-cases de baixa latência.

Vantagens: latência microsegundos, operações atômicas, persistência opcional. Limitações: dados na memória (custo), desenho para padrões de acesso que aproveitem velocidade em memória.

## <a id="instalacao-e-deployment">Instalação e deployment</a>

Instalação via pacote (Debian/Ubuntu):

```bash
sudo apt update
sudo apt install redis-server
```

Via Docker (rápido para testes):

```bash
docker run -d --name redis -p 6379:6379 redis:7
```

Para produção prefira imagens oficiais com configuração e volumes, ou pacotes da distro/compilação com TLS e ajustes.

## <a id="tipos-de-dados-e-comandos-essenciais">Tipos de dados e comandos essenciais</a>

- String: `SET key value`, `GET key`, `INCR`, `EXPIRE`
- Hash: `HSET user:1 name "Ana"`, `HGETALL user:1`
- List: `LPUSH queue item`, `BRPOP queue 0`
- Set: `SADD tags x`, `SMEMBERS tags`
- Sorted Set: `ZADD leaderboard 100 user1`, `ZRANGE`
- Streams: `XADD mystream * message "hello"`, `XREAD` — para filas duráveis
- Pub/Sub: `PUBLISH channel msg`, `SUBSCRIBE channel`

Use `redis-cli --stat` e `redis-cli --bigkeys` para diagnóstico.

## <a id="persistencia-rdb-vs-aof">Persistência: RDB vs AOF</a>

RDB (snapshots)
- Performa bem e gera arquivos compactos (`dump.rdb`). Bom para backups pontuais.
- RDB é crash-safe até o último snapshot (pode perder writes entre snapshots).

AOF (append-only file)
- Registra cada operação que altera dataset. Pode ser `appendfsync always/everysec/no`.
- Mais seguro para perda mínima de dados (especialmente com `everysec` ou `always`).

Combinação
- É comum usar RDB + AOF (AOF rewrite gera RDB-like snapshot internamente).

Configurações (trechos em `redis.conf`):

```conf
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
```

Backups e restore
- Fazer cópia do `dump.rdb` ou `appendonly.aof`. Para backup consistente, pare o Redis ou use `BGSAVE`/`BGREWRITEAOF` e copie os arquivos.
- Para restaurar, pare o servidor, substitua os arquivos em `dir` e inicie.

## <a id="replicacao-sentinel-e-cluster">Replicação, Sentinel e Cluster</a>

Replicação mestre->replica
- Habilite replicas conectando com `replicaof <masterip> <port>` ou `SLAVEOF` em versões antigas.
- Replicas são read-only por padrão (pode configurar `replica-read-only no`).

Sentinel (alta disponibilidade)
- Monitora master/replicas e realiza failover automático.
- Exemplo mínimo `sentinel.conf`: `sentinel monitor mymaster 127.0.0.1 6379 2`

Cluster (sharding nativo)
- Redis Cluster shard data entre nós via hash slots (0-16383). Use `redis-cli --cluster create` para criar cluster com múltiplos masters e réplicas.
- O cluster oferece disponibilidade e escalabilidade horizontal, porém algumas operações transacionais e multi-key precisam estar co-localizadas (mesmo hash slot).

## <a id="configuracao-e-tuning-de-memoria-eviction">Configuração e tuning de memória / eviction</a>

Memória
- `maxmemory` limita uso de RAM; combine com políticas de `maxmemory-policy` (noeviction, allkeys-lru, volatile-lru, etc).

Eviction policies
- `noeviction`: erros quando atinge limite.
- `allkeys-lru`/`volatile-lru`: LRU baseado em uso, `volatile-*` considera apenas chaves com TTL.

Exemplo `redis.conf`:

```conf
maxmemory 4gb
maxmemory-policy allkeys-lru
```

Perfis de latência
- Ajuste `tcp-keepalive`, `tcp-backlog`, `hz` e parâmetros do SO (net.core.somaxconn, vm.overcommit_memory).

## <a id="seguranca-e-autenticacao">Segurança e autenticação</a>

- Bind e firewall: `bind 127.0.0.1` ou proteja com firewall/ACLs.
- Autenticação: `requirepass <senha>` (antigo) e ACLs modernas (`ACL SETUSER`).
- Ativar TLS: compile com TLS ou use `stunnel` / proxy TLS.

Exemplo ACL:

```text
ACL SETUSER readonly on >secret ~* +@read
```

Boas práticas: segmente rede, não exponha Redis sem TLS/ACL, monitore e rote autenticação.

## <a id="monitoracao-e-operacoes">Monitoração e operações</a>

- `INFO` (server, memory, clients, persistence, stats, replication, cpu)
- `SLOWLOG` para queries lentas: `SLOWLOG GET 10`
- `MONITOR` para debug (consome CPU)
- Exporters: `redis_exporter` para Prometheus

Rotinas operacionais
- Teste `BGSAVE`/`BGREWRITEAOF` e monitore I/O.
- Monitorar AOF rewrite e RDB duration para evitar picos.

## <a id="scripting-lua-e-transactions">Scripting (Lua) e Transactions</a>

Lua scripts são atômicos e rodados no servidor (`EVAL`). Exemplo simples:

```lua
-- increment_if_exists.lua
local k = KEYS[1]
if redis.call('EXISTS', k) == 1 then
  return redis.call('INCR', k)
end
return nil
```

`MULTI`/`EXEC` oferecem transações simples; não garantem isolamento serializável entre clients.

## <a id="streams-e-patterns-de-consumo">Streams e patterns de consumo</a>

Redis Streams oferece modelo de log/consumer groups (XADD, XREADGROUP, XACK). Útil para filas duráveis e processamento distribuído.

## <a id="modulos-e-ecossistema">Módulos e ecossistema</a>

- RedisJSON, RediSearch, RedisGraph, RedisGears — estendem funcionalidades.
- Use módulos aprovados para casos específicos (full-text, consultas JSON, grafos).

## <a id="boas-praticas-pitfalls-e-scaling">Boas práticas, pitfalls e scaling</a>

- Evite operações O(N) em grandes chaves (e.g., KEYS * em produção).
- Prefira comandos iterativos (SCAN, SSCAN, HSCAN, ZSCAN).
- Particione dados com Cluster quando dataset ultrapassar memória de um nó.
- Teste induzindo falhas (Chaos) para validar failover e recovery.

## <a id="exemplos-rapidos-python">Exemplos rápidos (Python)</a>

```python
import redis
r = redis.Redis(host='127.0.0.1', port=6379, db=0)
r.set('foo', 'bar')
print(r.get('foo'))
```

## <a id="referencias">Referências</a>

- Redis official: https://redis.io/
- Redis commands: https://redis.io/commands
- Redis Cluster specification and tutorials no site oficial.

---

Este guia é um ponto de partida aprofundado; se quiser, eu posso:
- adicionar exemplos completos de configuração `redis.conf` para produção;  
- criar scripts de backup/restore automáticos;  
- incluir tutoriais passo-a-passo para criação de Cluster e Sentinel.  
