# Redis — Guia completo e aprofundado

Este documento cobre arquitetura interna, instalação, tipos de dados, persistência, replicação, clustering, tuning, segurança, operações, scripting, streams, pipelining, pub/sub, locks distribuídos, rate limiting, benchmarking, troubleshooting e exemplos práticos do Redis — do nível básico ao avançado.

## Índice

- [Introdução e arquitetura interna](#introducao-e-arquitetura-interna)
- [Casos de uso](#casos-de-uso)
- [Instalação e deployment](#instalacao-e-deployment)
- [Configuração base (redis.conf)](#configuracao-base)
- [Tipos de dados e comandos essenciais](#tipos-de-dados-e-comandos-essenciais)
- [Expiração, TTL e keyspace notifications](#expiracao-ttl-e-keyspace-notifications)
- [Persistência: RDB vs AOF](#persistencia-rdb-vs-aof)
- [Replicação, Sentinel e Cluster](#replicacao-sentinel-e-cluster)
- [Configuração e tuning de memória / eviction](#configuracao-e-tuning-de-memoria-eviction)
- [Pipelining e performance](#pipelining-e-performance)
- [Pub/Sub — padrões e limitações](#pubsub-padroes-e-limitacoes)
- [Streams e patterns de consumo](#streams-e-patterns-de-consumo)
- [Scripting (Lua), Functions e Transactions](#scripting-lua-functions-e-transactions)
- [Locks distribuídos (Redlock)](#locks-distribuidos-redlock)
- [Rate limiting](#rate-limiting)
- [Client-side caching (Redis 6+)](#client-side-caching)
- [Segurança e autenticação](#seguranca-e-autenticacao)
- [Monitoração e operações](#monitoracao-e-operacoes)
- [Benchmarking](#benchmarking)
- [Módulos e ecossistema](#modulos-e-ecossistema)
- [Boas práticas, pitfalls e scaling](#boas-praticas-pitfalls-e-scaling)
- [Troubleshooting](#troubleshooting)
- [Exemplos completos (Python)](#exemplos-completos-python)
- [Configuração redis.conf para produção](#configuracao-redis-conf-producao)
- [Referências](#referencias)

---

## <a id="introducao-e-arquitetura-interna">Introdução e arquitetura interna</a>

Redis (**RE**mote **DI**ctionary **S**erver) é um datastore em memória de código aberto, com suporte a estruturas de dados ricas (strings, hashes, lists, sets, sorted sets, streams, bitmaps, bitfields, HyperLogLog, geospatial indexes). Criado por Salvatore Sanfilippo em 2009, hoje é mantido pela Redis Ltd.

### Modelo single-threaded e I/O multiplexing

O Redis processa comandos em **uma única thread principal** usando um event loop baseado em I/O multiplexing (`epoll` no Linux, `kqueue` no macOS). Isso elimina a necessidade de locks e garante atomicidade em cada operação individual.

A partir do Redis 6, **I/O threading** foi adicionado para paralelizar leitura/escrita de socket (mas o processamento de comandos continua single-threaded):

```conf
# redis.conf — habilitar I/O threads (Redis 6+)
io-threads 4
io-threads-do-reads yes
```

### Arquitetura de memória

```text
┌────────────────────────────────────────────┐
│              Clients (TCP/TLS)             │
└──────────────────┬─────────────────────────┘
                   │ epoll / kqueue
┌──────────────────▼─────────────────────────┐
│           Event Loop (main thread)         │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐ │
│  │ Command  │  │ Keyspace │  │ Expire    │ │
│  │ Parser   │  │ Dict     │  │ Dict      │ │
│  └─────────┘  └──────────┘  └───────────┘ │
│  ┌─────────────────────────────────────┐   │
│  │  Persistence (RDB fork / AOF write) │   │
│  └─────────────────────────────────────┘   │
└────────────────────────────────────────────┘
```

- **Keyspace dict**: hash table principal que mapeia chaves para objetos.
- **Expire dict**: armazena TTLs de chaves com expiração.
- **Object encoding**: Redis escolhe internamente encoding otimizado (ziplist, intset, listpack, skiplist, hashtable) baseado no tamanho e tipo dos dados.

### Modelo de dados — databases

Redis suporta múltiplos databases numerados (padrão 16, de 0 a 15). Cada database é um namespace isolado:

```bash
redis-cli -n 3           # conecta ao db 3
SELECT 5                  # troca para db 5 na sessão corrente
```

> **Nota**: Em Redis Cluster, apenas o database 0 é suportado.

---

## <a id="casos-de-uso">Casos de uso</a>

| Caso de uso | Estrutura principal | Exemplo |
|---|---|---|
| **Cache de aplicação** | String / Hash | Cache de consultas SQL, respostas de API |
| **Sessões de usuário** | Hash | `HSET session:<token> user_id 42 expires_at ...` |
| **Filas de tarefas** | List / Stream | `LPUSH queue task` + `BRPOP queue 0` |
| **Rate limiting** | String + INCR | Contador por janela de tempo |
| **Leaderboards / Rankings** | Sorted Set | `ZADD leaderboard score player` |
| **Contadores em tempo real** | String (INCR) | Page views, likes, métricas |
| **Pub/Sub em tempo real** | Pub/Sub / Stream | Chat, notificações push |
| **Geolocalização** | Geospatial | `GEOADD locations lat lng name` |
| **Detecção de duplicatas** | Set / Bloom Filter | Deduplicação de eventos |
| **Counting único** | HyperLogLog | Visitantes únicos com `PFADD` |
| **Feature flags** | Hash / String | Controle de features por ambiente |
| **Locks distribuídos** | String + NX + PX | Mutex entre microserviços |
| **Primary datastore** | Hash / JSON | Dados de baixa latência (com persistência) |

---

## <a id="instalacao-e-deployment">Instalação e deployment</a>

### Via pacote (Debian/Ubuntu)

```bash
sudo apt update
sudo apt install redis-server
sudo systemctl enable redis-server
sudo systemctl start redis-server
redis-cli ping   # PONG
```

### Via pacote (RHEL/CentOS/Fedora)

```bash
sudo dnf install redis
sudo systemctl enable redis
sudo systemctl start redis
```

### Compilação a partir do código-fonte

```bash
sudo apt install build-essential tcl pkg-config libssl-dev
wget https://download.redis.io/redis-stable.tar.gz
tar xzf redis-stable.tar.gz
cd redis-stable
make BUILD_TLS=yes
make test
sudo make install
```

Isso instala `redis-server`, `redis-cli`, `redis-sentinel`, `redis-benchmark` e `redis-check-aof` em `/usr/local/bin/`.

### Via Docker

```bash
# Teste rápido
docker run -d --name redis -p 6379:6379 redis:7

# Produção com volume e config customizado
docker run -d \
  --name redis-prod \
  -p 6379:6379 \
  -v /data/redis:/data \
  -v /etc/redis/redis.conf:/usr/local/etc/redis/redis.conf \
  redis:7 redis-server /usr/local/etc/redis/redis.conf
```

### Docker Compose (com persistência)

```yaml
version: "3.8"
services:
  redis:
    image: redis:7
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 4g

volumes:
  redis_data:
    driver: local
```

### Kubernetes (StatefulSet básico)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "1"
        volumeMounts:
        - name: redis-data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  type: ClusterIP
  ports:
  - port: 6379
  selector:
    app: redis
```

> Para clusters Redis em Kubernetes, considere o **Bitnami Redis Helm Chart** ou o **Redis Operator** da Spotahome/Opstree.

### Validação pós-instalação

```bash
redis-cli ping                         # PONG
redis-cli INFO server | grep redis_version
redis-cli CONFIG GET maxmemory
redis-cli CONFIG GET save
```

---

## <a id="configuracao-base">Configuração base (redis.conf)</a>

Os parâmetros mais importantes para entender e ajustar:

```conf
# === REDE ===
bind 127.0.0.1 -::1          # interfaces de escuta (segurança!)
port 6379                      # porta padrão
protected-mode yes             # bloqueia conexões externas sem auth
tcp-backlog 511                # fila de conexões pendentes
timeout 0                      # timeout de idle (0 = desabilitado)
tcp-keepalive 300              # keepalive TCP em segundos

# === GERAL ===
daemonize no                   # sim se rodando sem systemd
pidfile /var/run/redis/redis-server.pid
loglevel notice                # debug | verbose | notice | warning
logfile /var/log/redis/redis-server.log
databases 16                   # número de databases (0-15)

# === MEMÓRIA ===
maxmemory 4gb
maxmemory-policy allkeys-lru

# === PERSISTÊNCIA (RDB) ===
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

# === PERSISTÊNCIA (AOF) ===
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# === SLOW LOG ===
slowlog-log-slower-than 10000  # microsegundos (10ms)
slowlog-max-len 128

# === CLIENTES ===
maxclients 10000
```

Para alterar configurações em runtime:

```bash
redis-cli CONFIG SET maxmemory 6gb
redis-cli CONFIG SET maxmemory-policy volatile-lfu
redis-cli CONFIG REWRITE   # salva alterações no redis.conf
```

---

## <a id="tipos-de-dados-e-comandos-essenciais">Tipos de dados e comandos essenciais</a>

### String

O tipo mais básico. Pode armazenar texto, números ou dados binários (até 512 MB).

```bash
SET user:name "Maria"                   # define valor
GET user:name                           # "Maria"
SET counter 10                          # número como string
INCR counter                            # 11 (atômico)
INCRBY counter 5                        # 16
DECR counter                            # 15
INCRBYFLOAT price 2.5                   # incremento float

# Com opções
SET session:abc "data" EX 3600          # expira em 3600s
SET session:abc "data" PX 60000         # expira em 60000ms
SET lock:resource "owner" NX EX 30      # set somente se não existe (lock)
SETNX key value                         # atalho para SET ... NX
SETEX key 60 value                      # set com expire em 1 comando

# Operações em lote
MSET k1 v1 k2 v2 k3 v3                 # múltiplos sets atômicos
MGET k1 k2 k3                          # múltiplos gets

# Manipulação de string
APPEND key " extra"                     # concatena
STRLEN key                              # tamanho em bytes
GETRANGE key 0 4                        # substring
SETRANGE key 6 "world"                  # substitui a partir da posição 6
GETSET key "new"                        # retorna antigo, define novo (deprecated, use GETDEL/GETEX)
GETDEL key                              # retorna e deleta
```

### Hash

Mapas campo-valor, ideais para representar objetos:

```bash
HSET user:1 name "Ana" email "ana@x.com" age 28
HGET user:1 name                        # "Ana"
HMGET user:1 name email                 # ["Ana", "ana@x.com"]
HGETALL user:1                          # todos os campos e valores
HDEL user:1 age                         # remove campo
HEXISTS user:1 email                    # 1 (existe)
HLEN user:1                             # número de campos
HINCRBY user:1 age 1                    # incrementa campo numérico
HINCRBYFLOAT user:1 score 1.5           # incremento float
HKEYS user:1                            # lista de campos
HVALS user:1                            # lista de valores
HSETNX user:1 role "admin"              # define campo somente se não existe
HSCAN user:1 0 MATCH "na*" COUNT 10     # iteração segura
```

**Encoding interno**: se o hash tem poucos campos e valores pequenos, Redis usa `listpack` (antigo `ziplist`) — mais eficiente em memória. Configura-se via:

```conf
hash-max-listpack-entries 128     # máximo de campos para listpack
hash-max-listpack-value 64        # tamanho máximo de valor (bytes)
```

### List

Listas ordenadas (linked list / quicklist):

```bash
RPUSH tasks "task1" "task2" "task3"     # insere à dir.
LPUSH tasks "task0"                     # insere à esq.
LRANGE tasks 0 -1                       # todos os elementos
LINDEX tasks 2                          # elemento na posição 2
LLEN tasks                              # tamanho
LPOP tasks                              # remove e retorna da esquerda
RPOP tasks                              # remove e retorna da direita
LPOS tasks "task2"                      # posição do elemento

# Bloqueante (ideal para filas)
BRPOP tasks 30                          # bloqueia até 30s esperando item
BLPOP tasks 30                          # idem, da esquerda
BRPOPLPUSH source dest 0               # pop + push atômico (deprecated)
BLMOVE source dest LEFT RIGHT 0         # substituto moderno (Redis 6.2+)
LMOVE source dest LEFT RIGHT            # mover sem bloqueio

# Trim (manter apenas N itens)
LTRIM tasks 0 99                        # mantém apenas os primeiros 100
LSET tasks 0 "updated_task"             # substitui elemento no índice 0
LINSERT tasks BEFORE "task2" "task1.5"  # insere antes de task2
LREM tasks 2 "task1"                    # remove 2 ocorrências de "task1"
```

**Padrão fila confiável** (reliable queue):

```bash
# Produtor
LPUSH queue:jobs '{"id":1,"action":"send_email"}'

# Consumidor (move para processing, depois confirma)
LMOVE queue:jobs queue:processing LEFT RIGHT
# ... processa ...
LREM queue:processing 1 '{"id":1,"action":"send_email"}'
```

### Set

Conjuntos não ordenados de strings únicas:

```bash
SADD fruits "apple" "banana" "cherry"
SMEMBERS fruits                         # todos os membros
SCARD fruits                            # cardinalidade (3)
SISMEMBER fruits "apple"                # 1 (é membro)
SMISMEMBER fruits "apple" "grape"       # [1, 0] (Redis 6.2+)
SREM fruits "banana"                    # remove
SPOP fruits                             # remove e retorna aleatório
SRANDMEMBER fruits 2                    # retorna 2 aleatórios sem remover

# Operações de conjunto
SADD setA "a" "b" "c"
SADD setB "b" "c" "d"
SINTER setA setB                        # {"b", "c"}
SUNION setA setB                        # {"a", "b", "c", "d"}
SDIFF setA setB                         # {"a"}
SINTERSTORE dest setA setB              # salva interseção em dest
SUNIONSTORE dest setA setB              # salva união em dest
SDIFFSTORE dest setA setB               # salva diferença em dest

# Iteração segura
SSCAN fruits 0 MATCH "a*" COUNT 100
```

### Sorted Set (ZSet)

Conjuntos ordenados por score (float):

```bash
ZADD leaderboard 100 "alice" 85 "bob" 92 "carol"
ZRANGE leaderboard 0 -1 WITHSCORES     # todas por score asc
ZREVRANGE leaderboard 0 -1 WITHSCORES  # desc
ZRANGEBYSCORE leaderboard 80 95        # score entre 80 e 95
ZRANGEBYLEX leaderboard "[a" "[d"      # range lexicográfico
ZRANK leaderboard "alice"              # posição (0-based, asc)
ZREVRANK leaderboard "alice"           # posição desc
ZSCORE leaderboard "alice"             # 100
ZCARD leaderboard                      # cardinalidade
ZINCRBY leaderboard 10 "bob"           # bob agora tem 95
ZREM leaderboard "carol"               # remove
ZCOUNT leaderboard 80 100              # contagem no range

# Operações entre sorted sets
ZINTERSTORE dest 2 zset1 zset2 WEIGHTS 1 2 AGGREGATE SUM
ZUNIONSTORE dest 2 zset1 zset2

# Pop por score
ZPOPMIN leaderboard                    # remove menor score
ZPOPMAX leaderboard                    # remove maior score
BZPOPMIN leaderboard 30                # bloqueante
BZPOPMAX leaderboard 30                # bloqueante

# Redis 6.2+ — ZRANGESTORE e ZRANGE unificado
ZRANGE leaderboard 0 -1 BYSCORE REV LIMIT 0 10 WITHSCORES

# Iteração segura
ZSCAN leaderboard 0 MATCH "a*" COUNT 100
```

**Encoding interno**: poucos elementos → `listpack`; muitos → `skiplist + hashtable`.

```conf
zset-max-listpack-entries 128
zset-max-listpack-value 64
```

### Bitmap

Operações bit a bit em strings:

```bash
SETBIT daily:logins:2026-02-28 1042 1    # user_id=1042 fez login
GETBIT daily:logins:2026-02-28 1042      # 1
BITCOUNT daily:logins:2026-02-28         # total de bits 1

# Operações entre bitmaps
BITOP AND result day1 day2               # AND entre dois bitmaps
BITOP OR result day1 day2                # OR
BITOP XOR result day1 day2              # XOR
BITOP NOT result day1                    # NOT

BITPOS daily:logins:2026-02-28 1         # posição do primeiro bit 1
BITPOS daily:logins:2026-02-28 0         # posição do primeiro bit 0

# BITFIELD — operações avançadas em campos inteiros dentro de strings
BITFIELD mykey SET u8 0 200              # define 8-bit unsigned na posição 0
BITFIELD mykey GET u8 0                  # lê
BITFIELD mykey INCRBY u8 0 10           # incrementa
```

**Caso de uso clássico**: rastreamento de atividade diária por user_id eficiente em memória (1 bit por usuário).

### HyperLogLog

Estimativa probabilística de cardinalidade (contagem de elementos únicos) com ~0.81% de erro e memória fixa de ~12 KB:

```bash
PFADD visitors:2026-02-28 "user1" "user2" "user3"
PFADD visitors:2026-02-28 "user1"        # duplicata ignorada
PFCOUNT visitors:2026-02-28              # ~3

# Merge de múltiplos HyperLogLogs
PFMERGE visitors:week visitors:2026-02-28 visitors:2026-02-27
PFCOUNT visitors:week
```

### Geospatial

Índices geoespaciais baseados em Sorted Sets:

```bash
GEOADD locations -43.1729 -22.9068 "Rio de Janeiro"
GEOADD locations -46.6333 -23.5505 "São Paulo"
GEOADD locations -38.5108 -12.9714 "Salvador"

GEODIST locations "Rio de Janeiro" "São Paulo" km      # distância em km
GEOPOS locations "Rio de Janeiro"                       # lat/lng

# Busca por raio (Redis < 6.2)
GEORADIUS locations -43.17 -22.90 200 km WITHCOORD WITHDIST COUNT 10 ASC

# Busca por raio (Redis 6.2+ — GEOSEARCH)
GEOSEARCH locations FROMLONLAT -43.17 -22.90 BYRADIUS 200 km ASC COUNT 10 WITHCOORD WITHDIST
GEOSEARCH locations FROMMEMBER "Rio de Janeiro" BYBOX 500 500 km ASC

# Salvar resultado
GEOSEARCHSTORE dest locations FROMLONLAT -43.17 -22.90 BYRADIUS 200 km ASC
```

### Comandos genéricos de chave

```bash
EXISTS key                      # 1 se existe
DEL key1 key2                   # deleta (bloqueante)
UNLINK key1 key2                # deleta assíncrono (não bloqueia)
TYPE key                        # tipo do valor
RENAME key newkey               # renomeia
RENAMENX key newkey             # renomeia somente se newkey não existe
COPY source dest                # copia (Redis 6.2+)
OBJECT ENCODING key             # encoding interno usado
OBJECT IDLETIME key             # tempo ocioso
OBJECT REFCOUNT key             # contagem de referências
DUMP key                        # serializa valor
RESTORE key 0 <serialized>     # restaura
RANDOMKEY                       # chave aleatória
DBSIZE                          # total de chaves no db
FLUSHDB [ASYNC]                 # apaga db corrente
FLUSHALL [ASYNC]                # apaga todos os dbs
```

### Iteração segura — SCAN

**Nunca use `KEYS *` em produção** — bloqueia o event loop. Use `SCAN`:

```bash
SCAN 0 MATCH "user:*" COUNT 100        # retorna cursor + chaves
SCAN <cursor> MATCH "user:*" COUNT 100  # continua iterando
# Parar quando cursor retornar 0

# Variantes por tipo de dado
HSCAN myhash 0 MATCH "field*" COUNT 100
SSCAN myset 0 MATCH "val*" COUNT 100
ZSCAN myzset 0 MATCH "mem*" COUNT 100
```

---

## <a id="expiracao-ttl-e-keyspace-notifications">Expiração, TTL e keyspace notifications</a>

### Configurando expiração

```bash
SET key "value"
EXPIRE key 3600                  # expira em 3600 segundos
PEXPIRE key 60000                # expira em 60000 milissegundos
EXPIREAT key 1772366400          # expira no timestamp Unix
PEXPIREAT key 1772366400000      # timestamp em ms

# Verificar TTL
TTL key                          # segundos restantes (-1 = sem expire, -2 = não existe)
PTTL key                         # milissegundos restantes

# Remover expiração
PERSIST key                      # torna permanente

# Redis 7+ — condições no EXPIRE
EXPIRE key 3600 NX               # set somente se não tem expire
EXPIRE key 3600 XX               # set somente se já tem expire
EXPIRE key 3600 GT               # set somente se novo TTL > atual
EXPIRE key 3600 LT               # set somente se novo TTL < atual
```

### Como a expiração funciona internamente

Redis usa duas estratégias combinadas:

1. **Lazy expiration**: quando uma chave expirada é acessada, Redis a detecta e remove.
2. **Active expiration**: a cada 100ms (configurável via `hz`), Redis testa amostras aleatórias de chaves com TTL e remove as expiradas. Se > 25% das amostras estão expiradas, repete imediatamente.

### Keyspace notifications

Permitem que clientes se inscrevam para receber eventos de operações no keyspace:

```conf
# redis.conf — habilitar notificações
notify-keyspace-events "Ex"     # E = keyevent, x = expired events
# Opções: K (keyspace), E (keyevent), g (generic), $ (string), l (list), 
#          s (set), h (hash), z (sorted set), x (expired), e (evicted), 
#          t (stream), m (key miss), A (alias para todos)
```

```bash
# Inscrever-se para eventos de expiração
redis-cli SUBSCRIBE '__keyevent@0__:expired'

# Em outro terminal
redis-cli SET temp "val" EX 5
# Após 5s, o subscriber recebe: "temp"
```

**Caso de uso**: invalidação de cache, limpeza de sessões expiradas, triggers.

> **Atenção**: keyspace notifications **não são confiáveis** em cluster e não garantem entrega (se o subscriber desconectar, perde eventos). Para filas confiáveis, use Streams.

---

## <a id="persistencia-rdb-vs-aof">Persistência: RDB vs AOF</a>

### RDB (Redis Database Backup — snapshots)

O Redis faz `fork()` do processo e o filho serializa o dataset inteiro para disco.

**Vantagens**:

- Arquivo compacto (`dump.rdb`), ideal para backups.
- Restauração muito rápida (load direto para memória).
- Performance mínima no processo principal (o filho faz o I/O).

**Desvantagens**:

- Pode perder dados entre snapshots (ex: últimos 5min).
- `fork()` pode ser lento com datasets grandes (Copy-on-Write do OS).

**Configuração**:

```conf
# Triggers de snapshot — "save <segundos> <nº mínimo de writes>"
save 900 1          # snapshot se ≥ 1 write em 900s
save 300 10         # snapshot se ≥ 10 writes em 300s
save 60 10000       # snapshot se ≥ 10000 writes em 60s
# Para desabilitar RDB:
# save ""

stop-writes-on-bgsave-error yes   # para writes se BGSAVE falhar
rdbcompression yes                 # compressão LZF
rdbchecksum yes                    # CRC64 para verificação de integridade
dbfilename dump.rdb
dir /var/lib/redis
```

**Comandos manuais**:

```bash
BGSAVE                  # snapshot em background (fork)
SAVE                    # snapshot bloqueante (NÃO use em produção)
LASTSAVE                # timestamp do último snapshot
DEBUG RELOAD            # recarrega RDB (debug)
```

### AOF (Append-Only File)

Cada comando de escrita é registrado em um log sequencial.

**Vantagens**:

- Perda mínima de dados (`always` = nenhuma, `everysec` = ~1s).
- Log legível e auditável.
- Rewrite automático mantém arquivo compacto.

**Desvantagens**:

- Arquivo maior que RDB.
- Restauração mais lenta (precisa replay de todos os comandos).
- `fsync` frequente pode impactar latência.

**Configuração**:

```conf
appendonly yes
appendfilename "appendonly.aof"
appenddirname "appendonlydir"     # Redis 7+ (diretório dedicado)

# Política de fsync
appendfsync always      # fsync a cada comando (mais seguro, mais lento)
appendfsync everysec    # fsync a cada segundo (recomendado)
appendfsync no          # deixa o OS decidir (mais rápido, menos seguro)

# Rewrite automático
no-appendfsync-on-rewrite no            # continua fsync durante rewrite
auto-aof-rewrite-percentage 100         # rewrite quando AOF dobra de tamanho
auto-aof-rewrite-min-size 64mb          # tamanho mínimo para trigger rewrite
aof-use-rdb-preamble yes                # Redis 4+ — AOF com preâmbulo RDB (mais rápido)
```

**AOF Multi-Part (Redis 7+)**: o AOF é dividido em base file (RDB preamble), incremental files e manifest:

```text
appendonlydir/
├── appendonly.aof.1.base.rdb       # base snapshot
├── appendonly.aof.1.incr.aof       # comandos incrementais
├── appendonly.aof.2.incr.aof       # próximo segmento
└── appendonly.aof.manifest         # manifesto
```

**Comandos manuais**:

```bash
BGREWRITEAOF                  # rewrite em background
redis-check-aof --fix <file>  # repara AOF corrompido
```

### Combinação RDB + AOF (recomendado para produção)

```conf
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes      # melhor dos dois mundos
```

Quando ambos estão habilitados e o Redis reinicia, ele prioriza o AOF para restauração (mais completo).

### Backups e restore

**Script de backup automatizado**:

```bash
#!/bin/bash
# backup_redis.sh — backup seguro do Redis
BACKUP_DIR="/backups/redis"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
REDIS_CLI="redis-cli -a ${REDIS_PASSWORD}"

mkdir -p "$BACKUP_DIR"

# Trigger snapshot
$REDIS_CLI BGSAVE
sleep 1

# Aguarda BGSAVE finalizar
while [ "$($REDIS_CLI LASTSAVE)" == "$LAST_SAVE" ]; do
    sleep 1
done

# Copia o RDB
REDIS_DIR=$($REDIS_CLI CONFIG GET dir | tail -1)
RDB_FILE=$($REDIS_CLI CONFIG GET dbfilename | tail -1)
cp "$REDIS_DIR/$RDB_FILE" "$BACKUP_DIR/dump_${TIMESTAMP}.rdb"

# Opcional: copia AOF
if [ -d "$REDIS_DIR/appendonlydir" ]; then
    cp -r "$REDIS_DIR/appendonlydir" "$BACKUP_DIR/aof_${TIMESTAMP}"
fi

# Rotação — mantém últimos 7 dias
find "$BACKUP_DIR" -name "dump_*.rdb" -mtime +7 -delete
find "$BACKUP_DIR" -name "aof_*" -type d -mtime +7 -exec rm -rf {} +

echo "Backup concluído: $BACKUP_DIR/dump_${TIMESTAMP}.rdb"
```

**Restore**:

```bash
sudo systemctl stop redis
cp /backups/redis/dump_20260228_120000.rdb /var/lib/redis/dump.rdb
sudo chown redis:redis /var/lib/redis/dump.rdb
sudo systemctl start redis
redis-cli DBSIZE    # verifica contagem de chaves
```

### Desabilitando persistência (cache puro)

```conf
save ""
appendonly no
```

> Útil quando o Redis é usado puramente como cache sem necessidade de durabilidade.

---

## <a id="replicacao-sentinel-e-cluster">Replicação, Sentinel e Cluster</a>

### Replicação Master → Replica

A replicação é assíncrona por padrão. A replica conecta, recebe um snapshot RDB e depois recebe um stream de comandos.

```text
┌──────────┐    replication    ┌──────────┐
│  Master  │ ────────────────► │ Replica 1│
│ (R/W)    │                   │ (R/O)    │
└──────────┘                   └──────────┘
      │          replication   ┌──────────┐
      └───────────────────────►│ Replica 2│
                               │ (R/O)    │
                               └──────────┘
```

**Configuração na replica** (`redis.conf`):

```conf
replicaof 192.168.1.10 6379
masterauth <senha-master>         # se master exige auth
replica-read-only yes              # padrão — replica somente leitura
replica-serve-stale-data yes       # responde mesmo durante sync inicial
repl-diskless-sync yes             # sync direto socket→replica (Redis 6+, bom para SSD lento)
repl-diskless-sync-delay 5         # delay em segundos antes de iniciar (espera mais replicas)
repl-backlog-size 256mb            # buffer de replicação (quanto dados manter para re-sync parcial)
repl-backlog-ttl 3600              # TTL do backlog em segundos
min-replicas-to-write 1            # escrita falha se < N replicas conectadas
min-replicas-max-lag 10            # lag máximo das replicas (segundos)
```

**Comandos úteis**:

```bash
INFO replication                    # status completo de replicação
REPLICAOF 192.168.1.10 6379        # torna instância replica (runtime)
REPLICAOF NO ONE                   # promove replica a master
WAIT 1 5000                        # espera ≥1 replica confirmar writes em até 5s
```

**Replicação em cadeia** (daisy-chain): uma replica pode ser master de outra replica, aliviando I/O do master principal.

### Sentinel (alta disponibilidade)

Redis Sentinel provê: monitoramento, notificação, failover automático e service discovery.

```text
┌───────────────────────────────────────────────┐
│                 Sentinel Quorum               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │Sentinel 1│ │Sentinel 2│ │Sentinel 3│      │
│  └─────┬────┘ └─────┬────┘ └─────┬────┘      │
│        │            │            │            │
│        ▼            ▼            ▼            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │  Master  │ │ Replica 1│ │ Replica 2│      │
│  └──────────┘ └──────────┘ └──────────┘      │
└───────────────────────────────────────────────┘
```

**Configuração** (`sentinel.conf`):

```conf
# Monitorar master — quorum de 2 sentinels para failover
sentinel monitor mymaster 192.168.1.10 6379 2

# Autenticação do master
sentinel auth-pass mymaster <senha>

# Tempo para considerar master down (ms)
sentinel down-after-milliseconds mymaster 5000

# Tempo máximo de failover
sentinel failover-timeout mymaster 60000

# Quantas replicas sincronizam simultaneamente durante failover
sentinel parallel-syncs mymaster 1

# Notificação (script executado em eventos)
sentinel notification-script mymaster /opt/redis/notify.sh

# Script executado após failover
sentinel client-reconfig-script mymaster /opt/redis/reconfig.sh
```

**Iniciar Sentinel**:

```bash
redis-sentinel /etc/redis/sentinel.conf
# ou
redis-server /etc/redis/sentinel.conf --sentinel
```

**Comandos do Sentinel**:

```bash
redis-cli -p 26379 SENTINEL masters                     # lista masters monitorados
redis-cli -p 26379 SENTINEL master mymaster              # info do master
redis-cli -p 26379 SENTINEL replicas mymaster            # lista replicas
redis-cli -p 26379 SENTINEL sentinels mymaster           # lista sentinels
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster  # IP:porta do master atual
redis-cli -p 26379 SENTINEL failover mymaster            # força failover manual
redis-cli -p 26379 SENTINEL reset mymaster               # reset monitoramento
```

**Layout de deploy recomendado**: mínimo 3 Sentinels em máquinas/zonas diferentes, com quorum = 2.

**Conectando com Sentinel em Python**:

```python
from redis.sentinel import Sentinel

sentinel = Sentinel(
    [('sentinel1', 26379), ('sentinel2', 26379), ('sentinel3', 26379)],
    socket_timeout=0.5
)

# Obtém conexão para master (escrita)
master = sentinel.master_for('mymaster', socket_timeout=0.5, password='secret')
master.set('key', 'value')

# Obtém conexão para replica (leitura)
replica = sentinel.slave_for('mymaster', socket_timeout=0.5, password='secret')
print(replica.get('key'))
```

### Cluster (sharding nativo)

Redis Cluster distribui dados por **16384 hash slots** entre múltiplos masters:

```text
Hash Slot = CRC16(key) mod 16384

┌──────────────────────────────────────────────────────┐
│                  Redis Cluster                       │
│                                                      │
│  Master A (slots 0-5460)     ◄── Replica A'          │
│  Master B (slots 5461-10922) ◄── Replica B'          │
│  Master C (slots 10923-16383)◄── Replica C'          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Criando um cluster (6 nós: 3 masters + 3 replicas)**:

```bash
# Iniciar 6 instâncias Redis com configuração de cluster
# Em cada redis.conf:
port 7000    # (7000-7005 para cada instância)
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 5000
appendonly yes

# Criar o cluster (após todos os nós iniciarem)
redis-cli --cluster create \
  192.168.1.10:7000 192.168.1.10:7001 \
  192.168.1.11:7000 192.168.1.11:7001 \
  192.168.1.12:7000 192.168.1.12:7001 \
  --cluster-replicas 1
```

**Operações de cluster**:

```bash
redis-cli -c -h 192.168.1.10 -p 7000    # -c = cluster mode (follow redirects)

# Info do cluster
CLUSTER INFO
CLUSTER NODES                             # todos os nós e slots
CLUSTER SLOTS                             # mapeamento slots → nós
CLUSTER MYID                              # ID do nó atual

# Administração
redis-cli --cluster check 192.168.1.10:7000       # health check
redis-cli --cluster info 192.168.1.10:7000         # resumo
redis-cli --cluster fix 192.168.1.10:7000          # tenta corrigir problemas
redis-cli --cluster reshard 192.168.1.10:7000      # mover slots entre nós
redis-cli --cluster rebalance 192.168.1.10:7000    # redistribuir slots uniformemente

# Adicionar nó
redis-cli --cluster add-node 192.168.1.13:7000 192.168.1.10:7000
redis-cli --cluster add-node 192.168.1.13:7001 192.168.1.10:7000 --cluster-slave

# Remover nó (primeiro migre slots)
redis-cli --cluster del-node 192.168.1.10:7000 <node-id>
```

**Hash tags** — forçar chaves no mesmo slot para multi-key operations:

```bash
# Tudo entre {} é usado para calcular o slot
SET {user:1}:name "Ana"
SET {user:1}:email "ana@x.com"
# Ambas ficam no mesmo slot → permite MGET, transações
MGET {user:1}:name {user:1}:email
```

**Limitações do Cluster**:

- Somente database 0.
- Multi-key operations (`MGET`, `MSET`, `SUNION` etc.) requerem que todas as chaves estejam no mesmo slot (use hash tags).
- `SELECT` não funciona.
- Pub/Sub funciona mas com broadcast para todos os nós (ineficiente para alta frequência).
- Scripts Lua e transações (`MULTI`/`EXEC`) devem operar em chaves do mesmo slot.

**Configuração de cluster avançada**:

```conf
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-migration-barrier 1              # mínimo replicas antes de migração
cluster-require-full-coverage yes        # cluster down se algum slot sem master
cluster-allow-reads-when-down no         # leitura quando cluster down?
cluster-allow-pubsubshard-when-down yes  # Redis 7+
```

---

## <a id="configuracao-e-tuning-de-memoria-eviction">Configuração e tuning de memória / eviction</a>

### Configuração de memória

```conf
maxmemory 4gb                    # limite de RAM
maxmemory-policy allkeys-lru     # política de eviction
maxmemory-samples 10             # amostras para algoritmo LRU/LFU (mais = mais preciso)
```

### Políticas de eviction detalhadas

| Política | Escopo | Algoritmo | Quando usar |
|---|---|---|---|
| `noeviction` | — | Retorna erro | Primary datastore (não pode perder dados) |
| `allkeys-lru` | Todas as chaves | LRU | Cache genérico (recomendado) |
| `allkeys-lfu` | Todas as chaves | LFU | Cache com dados "hot" frequentes |
| `allkeys-random` | Todas as chaves | Aleatório | Quando acesso é uniforme |
| `volatile-lru` | Chaves com TTL | LRU | Mix de dados cache + permanentes |
| `volatile-lfu` | Chaves com TTL | LFU | Similar, priorizando frequência |
| `volatile-random` | Chaves com TTL | Aleatório | Expiração simples |
| `volatile-ttl` | Chaves com TTL | Menor TTL | Prioriza remover o que expira antes |

**LRU vs LFU**:

- **LRU** (Least Recently Used): remove o que foi acessado há mais tempo. Bom para workloads genéricos.
- **LFU** (Least Frequently Used, Redis 4+): remove o que foi acessado menos vezes. Melhor para workloads com hot keys. Configurável via:

```conf
lfu-log-factor 10       # fator logarítmico (maior = incremento mais lento do contador)
lfu-decay-time 1        # decaimento em minutos (0 = sem decaimento)
```

### Análise de memória

```bash
INFO memory
# Campos importantes:
# used_memory              — memória alocada pelo Redis
# used_memory_rss          — memória residente real (do OS)
# used_memory_peak         — pico histórico
# mem_fragmentation_ratio  — RSS / used_memory (>1.5 = fragmentação alta)
# mem_allocator            — jemalloc, libc, etc.

MEMORY USAGE key                    # bytes usados por chave específica
MEMORY USAGE key SAMPLES 0          # análise exata (mais lento)
MEMORY DOCTOR                       # diagnóstico automático
MEMORY STATS                        # estatísticas detalhadas
MEMORY PURGE                        # solicita ao jemalloc liberar memória
MEMORY MALLOC-STATS                 # stats do allocator

redis-cli --bigkeys                 # encontra as maiores chaves por tipo
redis-cli --memkeys                 # similar, por memória
redis-cli --memkeys --samples 100   # com amostragem
```

### Otimização de memória

**1. Use encodings compactos**: Redis auto-seleciona encoding otimizado para estruturas pequenas:

```conf
# Thresholds para encoding compacto (listpack/ziplist)
hash-max-listpack-entries 128
hash-max-listpack-value 64
list-max-listpack-size -2        # -2 = 8KB por nó do quicklist
list-compress-depth 1            # compressão de nós internos (0 = desabilitado)
set-max-intset-entries 512       # intset para sets de inteiros
set-max-listpack-entries 128     # Redis 7.2+
zset-max-listpack-entries 128
zset-max-listpack-value 64
```

**2. Prefira hashes a muitas chaves individuais**: agrupar dados em hashes usa ~10x menos overhead.

```bash
# Ruim — 1000 chaves individuais
SET user:1:name "Ana"
SET user:1:email "ana@x.com"
# ... overhead de ~50 bytes por chave

# Bom — 1 hash com campos
HSET user:1 name "Ana" email "ana@x.com"
```

**3. Use short key names** (em ambientes com milhões de chaves):

```bash
# Em vez de:
SET user:preferences:theme "dark"
# Considere:
SET u:1:pref:theme "dark"
```

**4. Comprima valores grandes** no lado do cliente (gzip/lz4).

**5. Defragmentação ativa (Redis 4+)**:

```conf
activedefrag yes
active-defrag-enabled yes
active-defrag-ignore-bytes 100mb         # inicia defrag se frag ≥ 100MB
active-defrag-threshold-lower 10         # % mínimo de fragmentação
active-defrag-threshold-upper 100        # % máximo (esforço total)
active-defrag-cycle-min 1                # % CPU mínimo para defrag
active-defrag-cycle-max 25               # % CPU máximo para defrag
```

### Tuning do sistema operacional

```bash
# /etc/sysctl.conf — ajustes essenciais para Redis em produção

# Evita OOM killer e permite fork() eficiente
vm.overcommit_memory = 1

# Desabilita THP (Transparent Huge Pages) — causa latência no fork()
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Fila de conexões TCP
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# Keepalive TCP
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 5

# Limites de file descriptor
# /etc/security/limits.conf
redis soft nofile 65535
redis hard nofile 65535
```

**Systemd** — ajustar limites:

```ini
# /etc/systemd/system/redis.service.d/override.conf
[Service]
LimitNOFILE=65535
```

---

## <a id="pipelining-e-performance">Pipelining e performance</a>

### O que é pipelining

Pipelining permite enviar múltiplos comandos sem esperar a resposta de cada um, reduzindo drasticamente o round-trip time (RTT):

```text
Sem pipeline (3 comandos, 3 RTTs):
  Client ──SET──► Server    Client ◄──OK──  Server
  Client ──SET──► Server    Client ◄──OK──  Server
  Client ──SET──► Server    Client ◄──OK──  Server

Com pipeline (3 comandos, 1 RTT):
  Client ──SET──► 
         ──SET──► Server
         ──SET──►
  Client ◄──OK──
         ◄──OK── Server
         ◄──OK──
```

### Pipeline em Python

```python
import redis

r = redis.Redis(host='127.0.0.1', port=6379)

# Pipeline básico
pipe = r.pipeline(transaction=False)  # transaction=False = puro pipeline
for i in range(1000):
    pipe.set(f'key:{i}', f'value:{i}')
results = pipe.execute()  # envia tudo de uma vez

# Pipeline com transação (MULTI/EXEC)
pipe = r.pipeline(transaction=True)  # padrão
pipe.set('a', 1)
pipe.set('b', 2)
pipe.incr('a')
results = pipe.execute()  # [True, True, 2]

# Pipeline com leitura
pipe = r.pipeline(transaction=False)
for i in range(100):
    pipe.get(f'key:{i}')
results = pipe.execute()  # lista de 100 valores
```

### Batching inteligente

```python
# Para operações massivas, divida em lotes para evitar uso excessivo de memória
BATCH_SIZE = 1000

def bulk_set(r, data: dict):
    """Insere dados em lotes usando pipeline."""
    pipe = r.pipeline(transaction=False)
    count = 0
    for key, value in data.items():
        pipe.set(key, value)
        count += 1
        if count % BATCH_SIZE == 0:
            pipe.execute()
            pipe = r.pipeline(transaction=False)
    if count % BATCH_SIZE != 0:
        pipe.execute()
```

### Connection pooling

```python
import redis

# Pool de conexões (recomendado para aplicações multi-thread)
pool = redis.ConnectionPool(
    host='127.0.0.1',
    port=6379,
    db=0,
    max_connections=50,
    socket_timeout=5,
    socket_connect_timeout=5,
    retry_on_timeout=True,
    decode_responses=True,       # retorna str em vez de bytes
)
r = redis.Redis(connection_pool=pool)

# Pool com TLS
pool = redis.ConnectionPool(
    host='redis.example.com',
    port=6380,
    ssl=True,
    ssl_certfile='/path/to/client.crt',
    ssl_keyfile='/path/to/client.key',
    ssl_ca_certs='/path/to/ca.crt',
)
```

---

## <a id="pubsub-padroes-e-limitacoes">Pub/Sub — padrões e limitações</a>

### Pub/Sub básico

```bash
# Subscriber (terminal 1)
SUBSCRIBE news:tech news:science

# Publisher (terminal 2)
PUBLISH news:tech "Redis 8 released!"
# Retorna: número de subscribers que receberam

# Pattern subscribe
PSUBSCRIBE news:*              # recebe de todos os canais news:*

# Unsubscribe
UNSUBSCRIBE news:tech
PUNSUBSCRIBE news:*
```

### Pub/Sub em Python

```python
import redis
import threading

r = redis.Redis()

def subscriber():
    pubsub = r.pubsub()
    pubsub.subscribe('notifications')
    
    # Pattern subscribe
    pubsub.psubscribe('events:*')
    
    for message in pubsub.listen():
        if message['type'] in ('message', 'pmessage'):
            print(f"Canal: {message['channel']}, Dados: {message['data']}")

# Rodar subscriber em thread separada
t = threading.Thread(target=subscriber, daemon=True)
t.start()

# Publicar
r.publish('notifications', 'Novo pedido #123')
r.publish('events:login', 'user:42')
```

### Sharded Pub/Sub (Redis 7+)

Em cluster, Pub/Sub clássico faz broadcast para **todos** os nós. Sharded Pub/Sub roteia mensagens pelo hash slot do canal:

```bash
SSUBSCRIBE channel_name          # subscribe sharded
SPUBLISH channel_name "message"  # publish sharded
SUNSUBSCRIBE channel_name
```

### Limitações do Pub/Sub

| Limitação | Descrição |
|---|---|
| **Fire-and-forget** | Mensagens não são persistidas; se ninguém está ouvindo, a mensagem é perdida |
| **Sem replay** | Não há como recuperar mensagens anteriores |
| **Sem ACK** | Sem confirmação de processamento |
| **Buffer de saída** | Se subscriber é lento, Redis acumula buffer e pode desconectar o client |
| **Sem consumer groups** | Cada subscriber recebe todas as mensagens (fanout) |

> **Para filas duráveis com garantias de entrega, use Redis Streams.**

---

## <a id="streams-e-patterns-de-consumo">Streams e patterns de consumo</a>

Redis Streams (introduzidos no Redis 5) oferecem um log append-only persistente com consumer groups, similar ao Apache Kafka.

### Estrutura de um Stream

```text
Stream: mystream
┌──────────────────────────────────────────────────┐
│ 1677721600000-0  │ sensor="A" temp="22.5"        │
│ 1677721600001-0  │ sensor="B" temp="19.3"        │
│ 1677721600002-0  │ sensor="A" temp="23.1"        │
│ ...              │ ...                            │
└──────────────────────────────────────────────────┘
   ID (timestamp-seq)   Campos campo=valor
```

### Comandos básicos

```bash
# Adicionar entradas
XADD mystream * sensor "A" temp "22.5"          # * = ID automático
XADD mystream * sensor "B" temp "19.3"
XADD mystream 1677721600000-0 sensor "C" temp "20.0"  # ID manual

# Ler entradas
XLEN mystream                                     # total de entradas
XRANGE mystream - +                               # todas (- = menor, + = maior)
XRANGE mystream - + COUNT 10                      # primeiras 10
XRANGE mystream 1677721600000 +                   # a partir de timestamp
XREVRANGE mystream + - COUNT 5                    # últimas 5 (reverso)

# Leitura bloqueante
XREAD COUNT 10 BLOCK 5000 STREAMS mystream $      # $ = apenas novas entradas

# Leitura de múltiplos streams
XREAD COUNT 10 BLOCK 0 STREAMS stream1 stream2 $ $

# Info
XINFO STREAM mystream                             # metadados do stream
XINFO STREAM mystream FULL                        # info completa
XINFO GROUPS mystream                             # consumer groups

# Gerenciamento
XTRIM mystream MAXLEN 10000                       # mantém últimas 10000 entradas
XTRIM mystream MAXLEN ~ 10000                     # trim aproximado (mais rápido)
XTRIM mystream MINID 1677721600000-0              # remove entries com ID menor
XDEL mystream 1677721600000-0                     # deleta entrada específica
```

### Consumer Groups

Permitem que múltiplos consumidores processem o mesmo stream sem duplicação:

```bash
# Criar consumer group
XGROUP CREATE mystream mygroup $ MKSTREAM
# $ = começa a ler novas entries (0 = desde o início)

# Consumir (cada consumer recebe entries diferentes)
XREADGROUP GROUP mygroup consumer1 COUNT 5 BLOCK 2000 STREAMS mystream >
# > = somente entries não entregues a nenhum consumer do grupo

# Confirmar processamento (ACK)
XACK mystream mygroup 1677721600000-0

# Ver entries pendentes (não confirmadas)
XPENDING mystream mygroup                          # resumo
XPENDING mystream mygroup - + 10                   # detalhado
XPENDING mystream mygroup - + 10 consumer1         # de um consumer específico

# Reclamar entries de consumer inativo (failover de consumer)
XCLAIM mystream mygroup consumer2 3600000 1677721600000-0
# Reclama entry se idle > 3600000ms (1h)

# XAUTOCLAIM (Redis 6.2+) — claim automático de entries pendentes
XAUTOCLAIM mystream mygroup consumer2 3600000 0-0 COUNT 10

# Info do grupo
XINFO GROUPS mystream
XINFO CONSUMERS mystream mygroup

# Deletar consumer / grupo
XGROUP DELCONSUMER mystream mygroup consumer1
XGROUP DESTROY mystream mygroup
```

### Padrão completo em Python

```python
import redis
import time
import json

r = redis.Redis(host='127.0.0.1', decode_responses=True)

STREAM = 'events'
GROUP = 'event-processors'
CONSUMER = 'worker-1'

# Criar grupo (idempotente)
try:
    r.xgroup_create(STREAM, GROUP, id='0', mkstream=True)
except redis.exceptions.ResponseError as e:
    if 'BUSYGROUP' not in str(e):
        raise

def produce_events():
    """Produtor — adiciona eventos ao stream."""
    for i in range(100):
        r.xadd(STREAM, {
            'type': 'order',
            'order_id': str(i),
            'amount': str(i * 10.5),
            'timestamp': str(time.time())
        }, maxlen=100000)  # trim automático
        time.sleep(0.1)

def consume_events():
    """Consumidor com retry e ACK."""
    while True:
        try:
            # 1. Primeiro, processar entries pendentes (não ACK'd)
            pending = r.xpending_range(STREAM, GROUP, '-', '+', count=10, consumername=CONSUMER)
            if pending:
                ids = [entry['message_id'] for entry in pending]
                entries = r.xclaim(STREAM, GROUP, CONSUMER, min_idle_time=0, message_ids=ids)
                for entry_id, fields in entries:
                    process_event(entry_id, fields)
                    r.xack(STREAM, GROUP, entry_id)

            # 2. Ler novas entries
            results = r.xreadgroup(GROUP, CONSUMER, {STREAM: '>'}, count=10, block=5000)
            if results:
                for stream_name, entries in results:
                    for entry_id, fields in entries:
                        process_event(entry_id, fields)
                        r.xack(STREAM, GROUP, entry_id)

        except Exception as e:
            print(f"Erro: {e}")
            time.sleep(1)

def process_event(entry_id, fields):
    """Processa um evento."""
    print(f"Processando {entry_id}: {fields}")
    # ... lógica de negócio ...
```

### Stream vs Pub/Sub vs List

| Feature | Stream | Pub/Sub | List |
|---|---|---|---|
| Persistência | Sim | Não | Sim |
| Consumer groups | Sim | Não | Não |
| ACK/retry | Sim | Não | Manual |
| Replay histórico | Sim | Não | Parcial |
| Blocking read | Sim | Sim | Sim |
| Fanout | Sim (múltiplos groups) | Sim (nativo) | Não |
| Backpressure | Sim (XTRIM) | Buffer client | Não |

---

## <a id="scripting-lua-functions-e-transactions">Scripting (Lua), Functions e Transactions</a>

### Transactions (MULTI/EXEC)

Agrupa comandos que são executados atomicamente (nenhum outro comando se intercala):

```bash
MULTI                             # inicia transação
SET account:A 500
SET account:B 300
DECRBY account:A 100
INCRBY account:B 100
EXEC                              # executa tudo atomicamente
# Resultado: [OK, OK, 400, 400]

DISCARD                           # cancela transação pendente
```

**WATCH — optimistic locking**:

```bash
WATCH account:A                   # monitora chave
balance = GET account:A           # lê valor
MULTI
SET account:A (balance - 100)
EXEC
# Se account:A foi modificada entre WATCH e EXEC, EXEC retorna nil (falhou)
# Retry manualmente
```

**Limitações de MULTI/EXEC**:

- Sem rollback — se um comando falha, os outros executam.
- WATCH é otimista — pode falhar e precisa de retry.
- Não suporta lógica condicional (if/else) dentro da transação.

### Lua scripting (EVAL)

Scripts Lua rodam atomicamente no servidor, com acesso a comandos Redis e lógica condicional:

```bash
# Sintaxe: EVAL <script> <numkeys> <key1> ... <arg1> ...
EVAL "return redis.call('GET', KEYS[1])" 1 mykey

# Exemplo: transferência segura entre contas
EVAL "
local from = KEYS[1]
local to = KEYS[2]
local amount = tonumber(ARGV[1])

local balance = tonumber(redis.call('GET', from))
if balance == nil or balance < amount then
    return redis.error_reply('Saldo insuficiente')
end

redis.call('DECRBY', from, amount)
redis.call('INCRBY', to, amount)
return redis.status_reply('OK')
" 2 account:A account:B 100
```

**Rate limiter em Lua** (sliding window):

```lua
-- rate_limit.lua
-- KEYS[1] = chave do rate limit
-- ARGV[1] = limite máximo
-- ARGV[2] = janela em segundos
-- ARGV[3] = timestamp atual

local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Remove entries fora da janela
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- Conta requests na janela
local count = redis.call('ZCARD', key)

if count < limit then
    -- Adiciona request
    redis.call('ZADD', key, now, now .. '-' .. math.random(1000000))
    redis.call('EXPIRE', key, window)
    return 1  -- allowed
else
    return 0  -- denied
end
```

**Carregar e executar scripts (EVALSHA)**:

```bash
# Carregar script no servidor (retorna SHA1)
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# "e0e1f9fabfc9d4800c877a703b823ac0578ff831"

# Executar por SHA1 (mais eficiente que enviar script toda vez)
EVALSHA e0e1f9fabfc9d4800c877a703b823ac0578ff831 1 mykey

# Gerenciamento
SCRIPT EXISTS <sha1>       # verifica se script está em cache
SCRIPT FLUSH               # limpa cache de scripts
SCRIPT KILL                # mata script em execução (se não fez write)
```

**Em Python**:

```python
import redis

r = redis.Redis()

# Registrar script
transfer_script = r.register_script("""
local from = KEYS[1]
local to = KEYS[2]
local amount = tonumber(ARGV[1])
local balance = tonumber(redis.call('GET', from))
if balance == nil or balance < amount then
    return redis.error_reply('Saldo insuficiente')
end
redis.call('DECRBY', from, amount)
redis.call('INCRBY', to, amount)
return 1
""")

# Usar
result = transfer_script(keys=['account:A', 'account:B'], args=[100])
```

### Redis Functions (Redis 7+)

Functions substituem e melhoram EVAL/EVALSHA com bibliotecas nomeadas e persistentes:

```lua
-- Registrar uma biblioteca de functions
-- Salvar como mylib.lua e carregar com FUNCTION LOAD

#!lua name=mylib

-- Function: obter ou inicializar contador
redis.register_function('get_or_init', function(keys, args)
    local key = keys[1]
    local default = args[1] or '0'
    local val = redis.call('GET', key)
    if val == nil then
        redis.call('SET', key, default)
        return default
    end
    return val
end)

-- Function: transferir atomicamente
redis.register_function('transfer', function(keys, args)
    local from = keys[1]
    local to = keys[2]
    local amount = tonumber(args[1])
    
    local balance = tonumber(redis.call('GET', from) or '0')
    if balance < amount then
        return redis.error_reply('Saldo insuficiente')
    end
    
    redis.call('DECRBY', from, amount)
    redis.call('INCRBY', to, amount)
    return 'OK'
end)
```

```bash
# Carregar biblioteca
cat mylib.lua | redis-cli -x FUNCTION LOAD REPLACE

# Chamar function
FCALL get_or_init 1 counter "100"
FCALL transfer 2 account:A account:B 50

# Gerenciamento
FUNCTION LIST                  # listar bibliotecas
FUNCTION DUMP                  # exportar (binary)
FUNCTION RESTORE <dump>        # importar
FUNCTION DELETE mylib           # deletar biblioteca
FUNCTION FLUSH                  # limpar todas
FUNCTION STATS                  # estatísticas
```

**Vantagens de Functions sobre EVAL**:

- Nomeadas e persistentes (sobrevivem restart).
- Replicadas automaticamente para replicas.
- Melhor organização com bibliotecas.
- Flags de execução (no-writes, allow-oom, etc.).

---

## <a id="locks-distribuidos-redlock">Locks distribuídos (Redlock)</a>

### Lock simples (single instance)

```bash
# Adquirir lock
SET lock:resource <unique_id> NX PX 30000   # NX = somente se não existe, PX = TTL 30s

# Liberar lock (somente se sou o dono — via Lua)
EVAL "
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
" 1 lock:resource <unique_id>
```

### Redlock (multi-instance)

Para ambientes distribuí­dos, o algoritmo Redlock adquire o lock em N/2+1 instâncias independentes:

```python
from redis import Redis
from redis.lock import Lock
import uuid
import time

# Lock simples com redis-py
r = Redis()
lock = r.lock('lock:resource', timeout=30, blocking_timeout=10)

if lock.acquire():
    try:
        # ... operação protegida ...
        pass
    finally:
        lock.release()

# Ou com context manager
with r.lock('lock:resource', timeout=30, blocking_timeout=10):
    # ... operação protegida ...
    pass
```

**Implementação manual de Redlock com múltiplas instâncias**:

```python
import redis
import time
import uuid

class RedLock:
    def __init__(self, instances, resource, ttl=30000):
        self.instances = instances  # lista de redis.Redis
        self.resource = resource
        self.ttl = ttl
        self.quorum = len(instances) // 2 + 1
        self.lock_key = f"lock:{resource}"
        self.lock_value = str(uuid.uuid4())
    
    def acquire(self):
        start = time.monotonic()
        acquired = 0
        
        for instance in self.instances:
            try:
                if instance.set(self.lock_key, self.lock_value, nx=True, px=self.ttl):
                    acquired += 1
            except redis.exceptions.ConnectionError:
                continue
        
        elapsed = (time.monotonic() - start) * 1000
        validity = self.ttl - elapsed
        
        if acquired >= self.quorum and validity > 0:
            return True
        else:
            self.release()
            return False
    
    def release(self):
        release_script = """
        if redis.call('GET', KEYS[1]) == ARGV[1] then
            return redis.call('DEL', KEYS[1])
        end
        return 0
        """
        for instance in self.instances:
            try:
                instance.eval(release_script, 1, self.lock_key, self.lock_value)
            except redis.exceptions.ConnectionError:
                continue

# Uso
instances = [
    redis.Redis(host='redis1', port=6379),
    redis.Redis(host='redis2', port=6379),
    redis.Redis(host='redis3', port=6379),
]
lock = RedLock(instances, 'my-resource', ttl=30000)

if lock.acquire():
    try:
        # operação protegida
        pass
    finally:
        lock.release()
```

> **Atenção**: Redlock é debatido na comunidade (Martin Kleppmann vs Salvatore Sanfilippo). Para cenários críticos, considere usar um sistema de consenso como etcd ou ZooKeeper.

---

## <a id="rate-limiting">Rate limiting</a>

### Fixed window counter

```bash
# Limite: 100 requests por minuto
EVAL "
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = tonumber(redis.call('GET', key) or '0')
if current >= limit then
    return 0
end

if current == 0 then
    redis.call('SET', key, 1, 'EX', window)
else
    redis.call('INCR', key)
end
return 1
" 1 rate:api:user:42:202602281530 100 60
```

### Sliding window log (mais preciso)

```python
import redis
import time

r = redis.Redis()

def is_rate_limited(user_id: str, limit: int, window_seconds: int) -> bool:
    """Rate limiter com janela deslizante usando Sorted Set."""
    key = f"rate:{user_id}"
    now = time.time()
    window_start = now - window_seconds
    
    pipe = r.pipeline(transaction=True)
    pipe.zremrangebyscore(key, 0, window_start)   # remove entries antigas
    pipe.zcard(key)                                 # conta na janela
    pipe.zadd(key, {f"{now}": now})                # adiciona request atual
    pipe.expire(key, window_seconds)               # TTL de segurança
    results = pipe.execute()
    
    request_count = results[1]
    return request_count >= limit

# Uso
if is_rate_limited("user:42", limit=100, window_seconds=60):
    print("Rate limited!")
else:
    print("Allowed")
```

### Token bucket

```lua
-- token_bucket.lua
-- KEYS[1] = bucket key
-- ARGV[1] = max tokens (capacity)
-- ARGV[2] = refill rate (tokens por segundo)
-- ARGV[3] = tokens a consumir
-- ARGV[4] = timestamp atual (segundos float)

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local requested = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

-- Refill tokens
local elapsed = now - last_refill
local new_tokens = math.min(capacity, tokens + (elapsed * rate))

if new_tokens >= requested then
    new_tokens = new_tokens - requested
    redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
    return 1  -- allowed
else
    redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
    return 0  -- denied
end
```

---

## <a id="client-side-caching">Client-side caching (Redis 6+)</a>

Redis 6 introduziu **tracking** para client-side caching — o server notifica o client quando chaves cacheadas localmente são modificadas.

### Modo padrão (tracking)

```bash
# Habilitar tracking na conexão
CLIENT TRACKING ON

# Agora, qualquer GET que o client faça, Redis rastreia
GET user:1:name
# Se outro client alterar user:1:name, este client recebe invalidation message

# Desabilitar
CLIENT TRACKING OFF
```

### Modo broadcast

```bash
# Tracking por prefixo — recebe invalidação para todas as chaves que iniciam com "user:"
CLIENT TRACKING ON BCAST PREFIX user:
```

### Em Python (com hiredis)

```python
import redis

r = redis.Redis(protocol=3)  # RESP3 necessário para push messages
# Client-side caching é gerenciado automaticamente pelo driver
# com redis-py >= 5.0:
# r = redis.Redis(protocol=3, cache_enabled=True)
```

> O **RESP3** (Redis Serialization Protocol 3) é necessário para client-side caching funcionar nativamente. Certifique-se que o client suporta RESP3.

---

## <a id="seguranca-e-autenticacao">Segurança e autenticação</a>

### Bind e rede

```conf
# redis.conf — NUNCA exponha Redis na internet sem proteção
bind 127.0.0.1 -::1              # somente localhost
# bind 10.0.0.5                  # IP específico da rede interna
protected-mode yes                # bloqueia quando bind não é localhost e sem auth
port 6379
# port 0                         # desabilita TCP (somente Unix socket)
unixsocket /var/run/redis/redis.sock
unixsocketperm 770
```

### Autenticação legada (requirepass)

```conf
requirepass minha_senha_super_forte_123!
```

```bash
redis-cli
AUTH minha_senha_super_forte_123!
```

### ACLs (Redis 6+)

ACLs oferecem controle granular por usuário, comandos e chaves:

```bash
# Criar usuário com permissões específicas
ACL SETUSER app_user on >strong_password ~app:* +@all -@dangerous

# Sintaxe dos seletores:
# on/off           — habilita/desabilita usuário
# >password        — adiciona senha
# ~pattern         — chaves permitidas (glob)
# %R~pattern       — chaves somente leitura (Redis 7+)
# %W~pattern       — chaves somente escrita (Redis 7+)
# +command         — permite comando
# -command         — bloqueia comando
# +@category       — permite categoria de comandos
# -@category       — bloqueia categoria
# allcommands / nocommands
# allkeys / resetkeys
# allchannels / resetchannels   — controle Pub/Sub

# Exemplos práticos:
# Usuário read-only
ACL SETUSER reader on >reader_pass ~* +@read +@connection +ping

# Usuário para aplicação (sem comandos perigosos)
ACL SETUSER appuser on >app_pass ~app:* &app:* +@all -@admin -@dangerous -DEBUG -FLUSHALL -FLUSHDB -KEYS -SHUTDOWN

# Usuário admin
ACL SETUSER admin on >admin_pass ~* &* +@all

# Listar usuários e suas ACLs
ACL LIST
ACL GETUSER app_user

# Verificar quem sou
ACL WHOAMI

# Categorias de comandos disponíveis
ACL CAT                         # lista categorias
ACL CAT read                    # lista comandos na categoria 'read'

# Testar permissões
ACL DRYRUN appuser GET app:key1   # simula se o comando seria permitido

# Salvar ACLs em arquivo
ACL SAVE
ACL LOAD                         # recarrega do arquivo
```

**Arquivo de ACLs** (`/etc/redis/users.acl`):

```text
user default off
user admin on >admin_pass ~* &* +@all
user appuser on >app_pass ~app:* &app:* +@all -@admin -@dangerous
user reader on >reader_pass ~* +@read +@connection +PING +INFO
```

```conf
# redis.conf
aclfile /etc/redis/users.acl
```

### TLS/SSL

```conf
# redis.conf — TLS nativo (Redis 6+ compilado com BUILD_TLS=yes)
port 0                           # desabilita porta não-TLS
tls-port 6380
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-auth-clients optional       # 'yes' para mTLS obrigatório
tls-protocols "TLSv1.2 TLSv1.3"
tls-ciphers "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384"
tls-ciphersuites "TLS_AES_256_GCM_SHA384"
tls-prefer-server-ciphers yes

# Replicação com TLS
tls-replication yes

# Cluster com TLS
tls-cluster yes
```

**Gerar certificados (OpenSSL)**:

```bash
# CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/CN=Redis CA"

# Cert do servidor
openssl genrsa -out redis.key 2048
openssl req -new -key redis.key -out redis.csr -subj "/CN=redis.example.com"
openssl x509 -req -in redis.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out redis.crt -days 365 -sha256

# Cert do client (para mTLS)
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -subj "/CN=redis-client"
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256
```

**Conectar com TLS**:

```bash
redis-cli --tls --cert /path/client.crt --key /path/client.key --cacert /path/ca.crt -p 6380
```

### Checklist de segurança para produção

- [ ] `bind` restrito a interfaces internas
- [ ] `protected-mode yes`
- [ ] ACLs configuradas (desabilitar usuário `default`)
- [ ] TLS habilitado para conexões client e replicação
- [ ] Firewall limitando acesso à porta Redis
- [ ] `rename-command` para comandos perigosos (alternativa a ACLs):

```conf
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command DEBUG ""
rename-command CONFIG "CONFIG_2f8a3b"     # renomeia para string secreta
```

- [ ] Monitorar `ACL LOG` para tentativas de acesso negadas
- [ ] Redis não roda como root
- [ ] Backups encriptados
- [ ] Rotação de senhas e certificados

---

## <a id="monitoracao-e-operacoes">Monitoração e operações</a>

### Comando INFO detalhado

```bash
INFO                       # tudo
INFO server                # versão, uptime, OS
INFO clients               # conexões
INFO memory                # memória e fragmentação
INFO persistence           # RDB e AOF
INFO stats                 # comandos processados, keyspace hits/misses
INFO replication           # master/replica status
INFO cpu                   # consumo de CPU
INFO commandstats          # estatísticas por comando
INFO errorstats            # contagem de erros
INFO latencystats          # histograma de latência (Redis 7+)
INFO keyspace              # chaves por database
INFO modules               # módulos carregados
INFO all                   # tudo
```

**Métricas essenciais para monitorar**:

| Métrica | Comando/Campo | Alerta quando |
|---|---|---|
| Memória | `INFO memory → used_memory` | > 80% de `maxmemory` |
| Fragmentação | `INFO memory → mem_fragmentation_ratio` | > 1.5 ou < 1.0 |
| Conexões | `INFO clients → connected_clients` | > 80% de `maxclients` |
| Hit ratio | `INFO stats → keyspace_hits / (hits + misses)` | < 80% (cache) |
| Evictions | `INFO stats → evicted_keys` | Crescendo constantemente |
| Lag de replica | `INFO replication → master_repl_offset vs replica offset` | > 1MB ou crescendo |
| Slow log | `SLOWLOG LEN` | > 100 |
| Blocked clients | `INFO clients → blocked_clients` | > 0 por muito tempo |
| RDB duration | `INFO persistence → rdb_last_bgsave_time_sec` | > 60s |
| AOF rewrite | `INFO persistence → aof_last_rewrite_time_sec` | > 120s |

### SLOWLOG

```bash
# Configurar threshold
CONFIG SET slowlog-log-slower-than 10000   # 10ms em microsegundos
CONFIG SET slowlog-max-len 256

# Consultar
SLOWLOG GET 20              # últimas 20 queries lentas
SLOWLOG LEN                 # total de entries no log
SLOWLOG RESET               # limpa log

# Cada entry mostra:
# 1) ID
# 2) Timestamp
# 3) Duração (microsegundos)
# 4) Comando completo
# 5) IP:porta do client
# 6) Nome do client (se definido com CLIENT SETNAME)
```

### LATENCY (Redis 2.8.13+)

```bash
CONFIG SET latency-monitor-threshold 100   # monitora eventos > 100ms

LATENCY LATEST                # últimos eventos de latência
LATENCY HISTORY command       # histórico por tipo
LATENCY RESET                 # reset
LATENCY GRAPH command         # gráfico ASCII

# Tipos de eventos: fast-command, command, fork, aof-fsync, 
#                   rdb-unlink-temp-file, aof-write, etc.
```

### CLIENT

```bash
CLIENT LIST                    # lista todas as conexões
CLIENT LIST TYPE normal        # filtrar por tipo (normal, master, replica, pubsub)
CLIENT GETNAME                 # nome do client
CLIENT SETNAME "app-worker-1"  # define nome (útil para debug)
CLIENT ID                      # ID da conexão
CLIENT INFO                    # info da conexão atual
CLIENT KILL ID <id>            # desconecta client específico
CLIENT KILL ADDR 10.0.0.5:6379
CLIENT PAUSE 5000              # pausa todos os clients por 5s (útil para manutenção)
CLIENT UNPAUSE
CLIENT NO-EVICT on             # protege este client de eviction de client
```

### MONITOR (debug)

```bash
redis-cli MONITOR              # mostra todos os comandos em tempo real
# Saída:
# 1677721600.123456 [0 10.0.0.5:54321] "SET" "key" "value"
# 1677721600.123789 [0 10.0.0.5:54321] "GET" "key"
```

> **CUIDADO**: `MONITOR` consome CPU significativa (~50% de overhead). Use apenas para debug temporário.

### Prometheus + Grafana

**Redis Exporter** (oliver006/redis_exporter):

```bash
# Docker
docker run -d --name redis-exporter \
  -p 9121:9121 \
  -e REDIS_ADDR=redis://redis:6379 \
  -e REDIS_PASSWORD=secret \
  oliver006/redis_exporter:latest

# prometheus.yml
scrape_configs:
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

**Alertas Prometheus essenciais** (exemplo):

```yaml
groups:
  - name: redis
    rules:
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis instance down"
      
      - alert: RedisHighMemory
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage > 80%"
      
      - alert: RedisHighFragmentation
        expr: redis_mem_fragmentation_ratio > 1.5
        for: 10m
        labels:
          severity: warning
      
      - alert: RedisReplicationBroken
        expr: redis_connected_slaves < 1
        for: 1m
        labels:
          severity: critical
      
      - alert: RedisHighEvictions
        expr: rate(redis_evicted_keys_total[5m]) > 100
        for: 5m
        labels:
          severity: warning
      
      - alert: RedisTooManyConnections
        expr: redis_connected_clients / redis_config_maxclients > 0.8
        for: 5m
        labels:
          severity: warning
```

### Rotinas operacionais

```bash
# === Checklist diário ===
redis-cli INFO memory | grep -E "used_memory_human|mem_fragmentation_ratio"
redis-cli INFO persistence | grep -E "rdb_last_bgsave_status|aof_last_bgrewrite_status"
redis-cli INFO replication | grep -E "role|connected_slaves|master_repl_offset"
redis-cli SLOWLOG LEN
redis-cli DBSIZE

# === Análise de chaves ===
redis-cli --bigkeys                # encontra chaves grandes
redis-cli --memkeys --samples 1000 # análise de memória
redis-cli SCAN 0 MATCH "temp:*" COUNT 100  # procura chaves por padrão

# === Manutenção ===
redis-cli BGREWRITEAOF             # compactar AOF
redis-cli BGSAVE                   # forçar snapshot
redis-cli MEMORY PURGE             # liberar memória fragmentada
redis-cli CONFIG REWRITE           # salvar config alterada em runtime

# === Debug de conexões ===
redis-cli CLIENT LIST | grep -c "^"   # total de conexões
redis-cli CLIENT LIST | grep "idle=[0-9]\{4,\}"  # conexões idle > 1000s
```

---

## <a id="benchmarking">Benchmarking</a>

### redis-benchmark (built-in)

```bash
# Benchmark padrão
redis-benchmark -h 127.0.0.1 -p 6379 -c 50 -n 100000

# Opções comuns:
# -c <clients>      número de conexões paralelas (padrão: 50)
# -n <requests>     total de requests (padrão: 100000)
# -d <data_size>    tamanho do payload em bytes (padrão: 3)
# -t <tests>        testes específicos: set,get,lpush,etc
# -q                output resumido
# -P <pipeline>     número de comandos por pipeline
# --csv             output em CSV
# -a <password>     autenticação
# --threads <n>     threads para I/O (Redis 6+)

# Benchmark somente SET e GET com pipeline
redis-benchmark -t set,get -n 1000000 -P 16 -q

# Benchmark com payload grande
redis-benchmark -t set -n 100000 -d 1024 -q    # 1KB
redis-benchmark -t set -n 100000 -d 10240 -q   # 10KB

# Benchmark com script Lua
redis-benchmark -n 100000 EVAL "return redis.call('SET',KEYS[1],ARGV[1])" 1 bench:key value

# Benchmark realístico
redis-benchmark -c 100 -n 500000 -P 10 --threads 4 -t set,get,lpush,lpop,zadd,zpopmin -q
```

**Output exemplo**:

```text
SET: 187265.92 requests per second, p50=0.175 msec
GET: 196850.39 requests per second, p50=0.167 msec
LPUSH: 185873.61 requests per second, p50=0.183 msec
```

### memtier_benchmark (RedisLabs)

Mais sofisticado que redis-benchmark:

```bash
# Instalar
apt install memtier-benchmark

# Benchmark misto (1:10 set:get ratio)
memtier_benchmark \
  --server=127.0.0.1 --port=6379 \
  --clients=50 --threads=4 \
  --test-time=60 \
  --ratio=1:10 \
  --data-size=256 \
  --key-pattern=G:G \
  --key-maximum=1000000 \
  --pipeline=10

# Com TLS
memtier_benchmark --tls --cert client.crt --key client.key --cacert ca.crt
```

### Interpretando resultados

- **Throughput** (ops/sec): capacidade de processamento.
- **Latência p50/p99/p999**: percential de latência — o p99 é o mais importante para SLAs.
- **Pipeline**: aumentar pipeline reduz RTT overhead mas aumenta latência individual.
- **Comparar**: sempre benchmarke com configuração e hardware representativos da produção.

---

## <a id="modulos-e-ecossistema">Módulos e ecossistema</a>

### Módulos populares

| Módulo | Descrição | Exemplo |
|---|---|---|
| **RediSearch** | Full-text search, secondary indexes | `FT.SEARCH idx "hello world"` |
| **RedisJSON** | Documentos JSON nativos | `JSON.SET doc $ '{"name":"Ana"}'` |
| **RedisTimeSeries** | Séries temporais otimizadas | `TS.ADD sensor:temp * 22.5` |
| **RedisBloom** | Bloom Filter, Cuckoo, CMS, Top-K | `BF.ADD filter item` |
| **RedisGraph** | Banco de grafos (Cypher) | `GRAPH.QUERY g "MATCH (n) RETURN n"` |
| **RedisGears** | Processamento de eventos (triggers) | Execução de funções em eventos |
| **RedisAI** | Inferência ML (TensorFlow, PyTorch) | Scoring de modelos |
| **RedisCell** | Rate limiting (cell-rate algorithm) | `CL.THROTTLE key 15 30 60 1` |

### RedisJSON exemplo

```bash
# Carregar módulo
# redis.conf: loadmodule /path/to/rejson.so

JSON.SET user:1 $ '{"name":"Ana","age":28,"address":{"city":"SP"}}'
JSON.GET user:1 $                           # documento completo
JSON.GET user:1 $.name                      # "Ana"
JSON.SET user:1 $.age 29                    # atualizar campo
JSON.NUMINCRBY user:1 $.age 1              # incrementar
JSON.ARRAPPEND user:1 $.tags '"redis"'      # adicionar a array
JSON.DEL user:1 $.address                   # deletar campo
JSON.TYPE user:1 $                          # tipo
```

### RediSearch exemplo

```bash
# Criar índice
FT.CREATE idx:users ON HASH PREFIX 1 user: SCHEMA \
    name TEXT SORTABLE \
    email TAG \
    age NUMERIC SORTABLE \
    city TEXT

# Dados (hashes normais)
HSET user:1 name "Ana Silva" email "ana@x.com" age 28 city "São Paulo"
HSET user:2 name "Carlos Santos" email "carlos@x.com" age 35 city "Rio de Janeiro"

# Buscas
FT.SEARCH idx:users "Ana"                          # full-text
FT.SEARCH idx:users "@city:Paulo"                   # busca por campo
FT.SEARCH idx:users "@age:[25 35]"                  # range numérico
FT.SEARCH idx:users "@email:{ana@x.com}"            # tag exacta
FT.SEARCH idx:users "*" SORTBY age ASC LIMIT 0 10   # paginação

# Agregação
FT.AGGREGATE idx:users "*" \
    GROUPBY 1 @city \
    REDUCE COUNT 0 AS total \
    SORTBY 2 @total DESC
```

### RedisBloom (Bloom Filter)

```bash
# Probabilistic data structures
BF.ADD visited_urls "https://example.com"
BF.EXISTS visited_urls "https://example.com"    # 1 (provavelmente)
BF.EXISTS visited_urls "https://other.com"      # 0 (definitivamente não)

# Cuckoo Filter (suporta delete)
CF.ADD filter item1
CF.EXISTS filter item1    # 1
CF.DEL filter item1       # remove

# Count-Min Sketch (contagem aproximada)
CMS.INITBYPROB sketch 0.001 0.01
CMS.INCRBY sketch item1 5
CMS.QUERY sketch item1    # ~5

# Top-K
TOPK.ADD topk item1 item2 item3
TOPK.LIST topk
```

### RedisTimeSeries

```bash
# Criar time series
TS.CREATE sensor:temp RETENTION 86400000 LABELS location SP type temperature

# Adicionar dados
TS.ADD sensor:temp * 22.5
TS.ADD sensor:temp * 23.1

# Consultas
TS.GET sensor:temp                                          # último valor
TS.RANGE sensor:temp - + COUNT 100                          # range
TS.RANGE sensor:temp - + AGGREGATION avg 60000              # média por minuto
TS.MRANGE - + FILTER location=SP                            # multi-series por label

# Regras de downsampling automático
TS.CREATE sensor:temp:avg_1h LABELS location SP
TS.CREATERULE sensor:temp sensor:temp:avg_1h AGGREGATION avg 3600000
```

### Carregando módulos

```conf
# redis.conf
loadmodule /path/to/rejson.so
loadmodule /path/to/redisearch.so
loadmodule /path/to/redistimeseries.so
loadmodule /path/to/redisbloom.so
```

```bash
# Runtime
MODULE LOAD /path/to/module.so
MODULE LIST
MODULE UNLOAD module_name
```

> **Redis Stack** é uma distribuição que inclui RedisJSON, RediSearch, RedisTimeSeries e RedisBloom pré-instalados: `docker run -d --name redis-stack -p 6379:6379 redis/redis-stack:latest`

---

## <a id="boas-praticas-pitfalls-e-scaling">Boas práticas, pitfalls e scaling</a>

### Convenções de nomenclatura de chaves

```text
# Padrão recomendado: tipo:id:campo
user:1042:profile
session:abc123
cache:api:/v1/users?page=1
queue:emails
rate:api:user:42
lock:order:789
temp:import:batch:55        # chaves temporárias com TTL

# Separadores: use ":" (padrão da comunidade)
# Prefixos por serviço: svc1:user:1, svc2:user:1
```

### Pitfalls comuns

**1. `KEYS *` em produção**

```bash
# NUNCA faça isso — bloqueia o event loop por segundos/minutos
KEYS *              # O(N) — varre todo o keyspace
KEYS user:*         # idem

# USE SCAN
SCAN 0 MATCH "user:*" COUNT 100
```

**2. Hot keys**

Uma chave acessada milhares de vezes por segundo causa gargalo:

```text
Soluções:
- Read replicas para distribuir leitura
- Client-side caching (Redis 6+)
- Replicação da chave em múltiplas versões: key:1, key:2, ... key:N (sharding lógico)
- Cache local na aplicação com invalidação via Pub/Sub
```

**3. Big keys**

Chaves com valores muito grandes bloqueiam o event loop:

```bash
# Detectar
redis-cli --bigkeys
redis-cli MEMORY USAGE key

# Limites recomendados:
# String: < 1MB
# Hash/Set/ZSet/List: < 10.000 elementos (ou considere sharding)
```

**Solução — sharding de big keys**:

```python
# Em vez de um hash gigante:
# HSET users name1 "..." name2 "..." ... name100000 "..."

# Divida em buckets:
import hashlib

def get_bucket(field: str, num_buckets: int = 100) -> str:
    bucket = int(hashlib.md5(field.encode()).hexdigest(), 16) % num_buckets
    return f"users:{bucket}"

# HSET users:42 name1 "..." name2 "..."
# HSET users:73 name3 "..."
```

**4. Thundering herd (cache stampede)**

Quando uma chave de cache popular expira e muitos clientes tentam reconstruí-la simultaneamente:

```python
import redis
import time
import random

r = redis.Redis()

def get_with_stampede_protection(key, ttl, fetch_func):
    """Cache com proteção contra stampede usando lock."""
    value = r.get(key)
    if value is not None:
        return value
    
    lock_key = f"lock:{key}"
    # Tenta adquirir lock para rebuild
    if r.set(lock_key, "1", nx=True, ex=30):
        try:
            value = fetch_func()
            r.set(key, value, ex=ttl)
            return value
        finally:
            r.delete(lock_key)
    else:
        # Outro processo está reconstruindo — espera
        for _ in range(50):
            time.sleep(0.1)
            value = r.get(key)
            if value is not None:
                return value
        # Fallback: constrói sem lock
        return fetch_func()

# Alternativa: probabilistic early expiration
def get_with_early_expiration(key, ttl, fetch_func, beta=1.0):
    """XFetch algorithm — renova cache probabilisticamente antes de expirar."""
    pipe = r.pipeline()
    pipe.get(key)
    pipe.pttl(key)
    value, remaining_ttl = pipe.execute()
    
    if value is not None and remaining_ttl > 0:
        # Probabilidade de renovar = exp(-remaining_ttl / (beta * ttl))
        import math
        delta = ttl * 1000  # em ms
        if remaining_ttl < delta * 0.1:  # últimos 10% do TTL
            if random.random() < math.exp(-remaining_ttl / (beta * delta)):
                # Renova proativamente
                value = fetch_func()
                r.set(key, value, px=int(ttl * 1000))
        return value
    
    value = fetch_func()
    r.set(key, value, px=int(ttl * 1000))
    return value
```

**5. Não definir `maxmemory` em produção**

Sem limite, Redis consome toda a RAM e o OOM killer mata o processo:

```conf
# SEMPRE defina em produção
maxmemory 4gb
maxmemory-policy allkeys-lru
```

**6. Usar SELECT com múltiplos databases**

```text
Problemas:
- Thread única = contention entre databases
- Cluster não suporta (só db 0)
- Monitorar e debugar múltiplos dbs é mais difícil

Preferência: use prefixos de chave ou instâncias Redis separadas.
```

### Scaling

**Vertical** (scale up):

- Mais RAM, CPU mais rápido, SSD NVMe para persistência.
- Redis é single-threaded para comandos, então CPU clock importa mais que cores.
- Limite prático: ~25GB RAM / ~100K ops/sec por instância.

**Horizontal** (scale out):

```text
Estratégia 1: Read replicas
  - Distribui leitura entre master e replicas
  - Bom para workloads read-heavy (>80% reads)

Estratégia 2: Redis Cluster
  - Sharding automático por hash slots
  - Escala writes e reads
  - Requer hash tags para multi-key ops

Estratégia 3: Client-side sharding
  - Aplicação decide para qual instância enviar
  - Mais controle, mais complexidade

Estratégia 4: Proxy (Twemproxy, Redis Cluster Proxy)
  - Proxy transparente que distribui entre instâncias
  - Simplifica clients
```

**Padrão de cache multi-camada**:

```text
Request → Cache L1 (in-process, ms) 
        → Cache L2 (Redis, ~1ms)
        → Database (10-100ms)
```

---

## <a id="troubleshooting">Troubleshooting</a>

### Problemas comuns e soluções

**1. Redis está lento**

```bash
# Verificar slowlog
SLOWLOG GET 20

# Verificar latência
redis-cli --latency              # latência básica
redis-cli --latency-history      # histórico
redis-cli --latency-dist         # distribuição ASCII
redis-cli --intrinsic-latency 5  # latência intrínseca do sistema (5s teste)

CONFIG SET latency-monitor-threshold 50
LATENCY LATEST

# Verificar se fork está lento (para BGSAVE/BGREWRITEAOF)
INFO persistence | grep rdb_last_bgsave_time_sec

# Verificar comandos O(N)
INFO commandstats | grep -E "keys|hgetall|smembers|lrange"

# Verificar se THP está habilitado
cat /sys/kernel/mm/transparent_hugepage/enabled
# Se [always], desabilite: echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

**2. Memória alta / OOM**

```bash
INFO memory
MEMORY DOCTOR
redis-cli --bigkeys
redis-cli --memkeys

# Verificar fragmentação
# mem_fragmentation_ratio > 1.5 → fragmentação alta
# Solução: habilitar active defrag ou restart

# Verificar evictions
INFO stats | grep evicted_keys
```

**3. Conexões recusadas / timeout**

```bash
INFO clients
# connected_clients vs maxclients

# Verificar file descriptors
cat /proc/$(pidof redis-server)/limits | grep "open files"
# Aumentar se necessário

# Verificar backlog
INFO server | grep tcp_backlog
# Verificar net.core.somaxconn do OS

# Clients bloqueados
CLIENT LIST | grep blocked
```

**4. Replicação quebrada**

```bash
# No master
INFO replication
# Verificar: connected_slaves, master_repl_offset

# Na replica
INFO replication
# Verificar: master_link_status (deve ser "up")
# master_last_io_seconds_ago (deve ser < 10)
# master_sync_in_progress (0 = ok)

# Se repl_backlog_size for muito pequeno, replicas podem pedir full sync
# Solução: aumentar repl-backlog-size
CONFIG SET repl-backlog-size 512mb
```

**5. Cluster em estado inconsistente**

```bash
redis-cli --cluster check 127.0.0.1:7000
redis-cli --cluster fix 127.0.0.1:7000
CLUSTER INFO    # cluster_state deve ser "ok"
CLUSTER NODES   # verificar se todos os nós e slots estão corretos
```

**6. AOF corrompido**

```bash
# Verificar e reparar
redis-check-aof --fix /var/lib/redis/appendonlydir/appendonly.aof.1.incr.aof

# Se necessário, restaurar de backup RDB
sudo systemctl stop redis
cp /backups/redis/dump_latest.rdb /var/lib/redis/dump.rdb
rm -rf /var/lib/redis/appendonlydir
sudo systemctl start redis
```

### Debug avançado

```bash
# Memory analysis detalhado
redis-cli MEMORY USAGE key SAMPLES 0    # custo exato da chave
redis-cli DEBUG OBJECT key               # info interna do objeto
redis-cli OBJECT ENCODING key            # encoding usado
redis-cli OBJECT FREQ key                # frequência de acesso (LFU)
redis-cli OBJECT IDLETIME key            # tempo idle (LRU)
redis-cli OBJECT HELP                    # help

# Dump e análise offline (redis-rdb-tools)
pip install rdbtools python-lzf
rdb --command memory /var/lib/redis/dump.rdb --bytes 1024 -f memory.csv
# Analisa chaves > 1KB

# ACL log (tentativas negadas)
ACL LOG 10
ACL LOG RESET
```

---

## <a id="exemplos-completos-python">Exemplos completos (Python)</a>

### Setup básico

```python
import redis

# Conexão simples
r = redis.Redis(host='127.0.0.1', port=6379, db=0, decode_responses=True)
r.ping()  # True

# Com URL
r = redis.from_url('redis://:password@localhost:6379/0', decode_responses=True)

# Com TLS
r = redis.Redis(
    host='redis.example.com', port=6380,
    ssl=True, ssl_certfile='client.crt',
    ssl_keyfile='client.key', ssl_ca_certs='ca.crt',
    decode_responses=True
)
```

### Async (aioredis / redis-py 4.2+)

```python
import asyncio
import redis.asyncio as aioredis

async def main():
    r = aioredis.from_url('redis://localhost', decode_responses=True)
    
    await r.set('key', 'value')
    value = await r.get('key')
    print(value)  # 'value'
    
    # Pipeline async
    async with r.pipeline(transaction=False) as pipe:
        for i in range(100):
            pipe.set(f'async:{i}', i)
        await pipe.execute()
    
    # Pub/Sub async
    pubsub = r.pubsub()
    await pubsub.subscribe('channel')
    
    async for message in pubsub.listen():
        if message['type'] == 'message':
            print(message['data'])
            break
    
    await r.aclose()

asyncio.run(main())
```

### Cache decorator

```python
import redis
import json
import functools
import hashlib

r = redis.Redis(decode_responses=True)

def redis_cache(ttl=300, prefix="cache"):
    """Decorator para cachear resultado de funções no Redis."""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # Gerar chave baseada na função e argumentos
            key_data = f"{func.__module__}.{func.__qualname__}:{args}:{sorted(kwargs.items())}"
            cache_key = f"{prefix}:{hashlib.sha256(key_data.encode()).hexdigest()[:16]}"
            
            # Tentar cache
            cached = r.get(cache_key)
            if cached is not None:
                return json.loads(cached)
            
            # Executar função
            result = func(*args, **kwargs)
            
            # Salvar no cache
            r.set(cache_key, json.dumps(result, default=str), ex=ttl)
            return result
        
        wrapper.invalidate = lambda *args, **kwargs: r.delete(
            f"{prefix}:{hashlib.sha256(f'{func.__module__}.{func.__qualname__}:{args}:{sorted(kwargs.items())}'.encode()).hexdigest()[:16]}"
        )
        return wrapper
    return decorator

# Uso
@redis_cache(ttl=600)
def get_user_profile(user_id: int) -> dict:
    """Busca perfil do banco — cacheado por 10 min."""
    # ... query ao banco ...
    return {"id": user_id, "name": "Ana"}

profile = get_user_profile(42)  # cache miss → busca no banco
profile = get_user_profile(42)  # cache hit
get_user_profile.invalidate(42) # invalida cache
```

### Session manager

```python
import redis
import uuid
import json
import time

class RedisSessionManager:
    def __init__(self, redis_client, prefix="session", ttl=3600):
        self.r = redis_client
        self.prefix = prefix
        self.ttl = ttl
    
    def create(self, user_id: int, data: dict = None) -> str:
        """Cria nova sessão e retorna token."""
        token = str(uuid.uuid4())
        key = f"{self.prefix}:{token}"
        session_data = {
            "user_id": user_id,
            "created_at": time.time(),
            **(data or {})
        }
        self.r.hset(key, mapping={k: json.dumps(v) if isinstance(v, (dict, list)) else str(v) 
                                   for k, v in session_data.items()})
        self.r.expire(key, self.ttl)
        return token
    
    def get(self, token: str) -> dict | None:
        """Retorna dados da sessão ou None."""
        key = f"{self.prefix}:{token}"
        data = self.r.hgetall(key)
        if not data:
            return None
        self.r.expire(key, self.ttl)  # renovar TTL
        return data
    
    def update(self, token: str, fields: dict):
        """Atualiza campos da sessão."""
        key = f"{self.prefix}:{token}"
        if self.r.exists(key):
            self.r.hset(key, mapping={k: str(v) for k, v in fields.items()})
            self.r.expire(key, self.ttl)
    
    def destroy(self, token: str):
        """Destrói sessão."""
        self.r.delete(f"{self.prefix}:{token}")
    
    def count_active(self) -> int:
        """Conta sessões ativas (cuidado com performance em scale)."""
        count = 0
        cursor = 0
        while True:
            cursor, keys = self.r.scan(cursor, match=f"{self.prefix}:*", count=1000)
            count += len(keys)
            if cursor == 0:
                break
        return count

# Uso
r = redis.Redis(decode_responses=True)
sessions = RedisSessionManager(r, ttl=7200)

token = sessions.create(user_id=42, data={"role": "admin"})
session = sessions.get(token)
sessions.update(token, {"last_page": "/dashboard"})
sessions.destroy(token)
```

### Job queue simples

```python
import redis
import json
import time
import uuid

class SimpleJobQueue:
    def __init__(self, redis_client, queue_name="jobs"):
        self.r = redis_client
        self.queue = f"queue:{queue_name}"
        self.processing = f"queue:{queue_name}:processing"
        self.failed = f"queue:{queue_name}:failed"
    
    def enqueue(self, job_type: str, payload: dict, priority: int = 0) -> str:
        """Adiciona job à fila."""
        job_id = str(uuid.uuid4())
        job = json.dumps({
            "id": job_id,
            "type": job_type,
            "payload": payload,
            "created_at": time.time(),
            "attempts": 0
        })
        if priority > 0:
            self.r.lpush(self.queue, job)   # alta prioridade → início
        else:
            self.r.rpush(self.queue, job)   # normal → final
        return job_id
    
    def dequeue(self, timeout=30) -> dict | None:
        """Consome job com move atômico para lista de processing."""
        result = self.r.blmove(self.queue, self.processing, timeout, "LEFT", "RIGHT")
        if result:
            return json.loads(result)
        return None
    
    def complete(self, job: dict):
        """Marca job como completo."""
        self.r.lrem(self.processing, 1, json.dumps(job))
    
    def fail(self, job: dict, error: str):
        """Move job para fila de falhas."""
        job["error"] = error
        job["failed_at"] = time.time()
        pipe = self.r.pipeline()
        pipe.lrem(self.processing, 1, json.dumps({k: v for k, v in job.items() 
                                                    if k not in ("error", "failed_at")}))
        pipe.rpush(self.failed, json.dumps(job))
        pipe.execute()
    
    def stats(self) -> dict:
        """Estatísticas da fila."""
        return {
            "pending": self.r.llen(self.queue),
            "processing": self.r.llen(self.processing),
            "failed": self.r.llen(self.failed),
        }

# Uso — Produtor
r = redis.Redis(decode_responses=True)
queue = SimpleJobQueue(r)
queue.enqueue("send_email", {"to": "user@x.com", "subject": "Hello"})

# Uso — Worker
while True:
    job = queue.dequeue(timeout=30)
    if job:
        try:
            print(f"Processando: {job}")
            # ... executa job ...
            queue.complete(job)
        except Exception as e:
            queue.fail(job, str(e))
```

### Leaderboard completo

```python
import redis

class Leaderboard:
    def __init__(self, redis_client, name="leaderboard"):
        self.r = redis_client
        self.key = f"lb:{name}"
    
    def add_score(self, player: str, score: float):
        """Adiciona ou atualiza score do jogador."""
        self.r.zadd(self.key, {player: score})
    
    def increment_score(self, player: str, amount: float):
        """Incrementa score."""
        return self.r.zincrby(self.key, amount, player)
    
    def get_rank(self, player: str) -> int | None:
        """Retorna posição (1-based, desc)."""
        rank = self.r.zrevrank(self.key, player)
        return rank + 1 if rank is not None else None
    
    def get_score(self, player: str) -> float | None:
        """Retorna score do jogador."""
        return self.r.zscore(self.key, player)
    
    def top(self, n=10) -> list[tuple[str, float]]:
        """Top N jogadores."""
        return self.r.zrevrange(self.key, 0, n - 1, withscores=True)
    
    def around_me(self, player: str, n=5) -> list[tuple[str, float]]:
        """Jogadores ao redor da posição do player."""
        rank = self.r.zrevrank(self.key, player)
        if rank is None:
            return []
        start = max(0, rank - n)
        end = rank + n
        return self.r.zrevrange(self.key, start, end, withscores=True)
    
    def total_players(self) -> int:
        return self.r.zcard(self.key)
    
    def remove(self, player: str):
        self.r.zrem(self.key, player)

# Uso
r = redis.Redis(decode_responses=True)
lb = Leaderboard(r, "game:season1")
lb.add_score("alice", 1500)
lb.add_score("bob", 1200)
lb.increment_score("bob", 350)
print(lb.top(10))
print(f"Alice está em #{lb.get_rank('alice')}")
```

---

## <a id="configuracao-redis-conf-producao">Configuração redis.conf para produção</a>

```conf
# ============================================================
# redis.conf — Configuração de produção completa e comentada
# ============================================================

# --- REDE ---
bind 10.0.0.5                           # IP da rede interna (NUNCA 0.0.0.0)
port 6379
protected-mode yes
tcp-backlog 65535
timeout 300                              # timeout de idle 5min
tcp-keepalive 300

# --- TLS (descomente para habilitar) ---
# port 0
# tls-port 6380
# tls-cert-file /etc/redis/tls/redis.crt
# tls-key-file /etc/redis/tls/redis.key
# tls-ca-cert-file /etc/redis/tls/ca.crt
# tls-auth-clients optional
# tls-protocols "TLSv1.2 TLSv1.3"

# --- GERAL ---
daemonize no                             # no se usando systemd
supervised systemd                       # notifica systemd
pidfile /var/run/redis/redis-server.pid
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16
always-show-logo no
set-proc-title yes
proc-title-template "{title} {laddr} {server-mode}"

# --- I/O THREADING (Redis 6+) ---
io-threads 4                             # ajustar para nº de cores disponíveis
io-threads-do-reads yes

# --- MEMÓRIA ---
maxmemory 4gb                            # 75% da RAM total disponível
maxmemory-policy allkeys-lru             # ou allkeys-lfu para workloads com hot keys
maxmemory-samples 10
# lazyfree para não bloquear event loop em deletes grandes
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
lazyfree-lazy-user-del yes
lazyfree-lazy-user-flush yes

# --- PERSISTÊNCIA RDB ---
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis
rdb-del-sync-files no

# --- PERSISTÊNCIA AOF ---
appendonly yes
appendfilename "appendonly.aof"
appenddirname "appendonlydir"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 256mb          # aumentado para reduzir rewrites
aof-use-rdb-preamble yes
aof-timestamp-enabled no

# --- CLIENTES ---
maxclients 10000

# --- SLOW LOG ---
slowlog-log-slower-than 10000            # 10ms
slowlog-max-len 256

# --- LATENCY MONITOR ---
latency-monitor-threshold 100            # 100ms

# --- ACL ---
aclfile /etc/redis/users.acl
# acllog-max-len 128

# --- ENCODING OTIMIZADO ---
hash-max-listpack-entries 128
hash-max-listpack-value 64
list-max-listpack-size -2
list-compress-depth 1
set-max-intset-entries 512
set-max-listpack-entries 128
zset-max-listpack-entries 128
zset-max-listpack-value 64

# --- DEFRAGMENTAÇÃO ATIVA ---
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100
active-defrag-cycle-min 1
active-defrag-cycle-max 25
active-defrag-max-scan-fields 1000

# --- SEGURANÇA ---
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command DEBUG ""
# rename-command CONFIG "CONFIG_secret_2f8a3b"

# --- REPLICAÇÃO (se for master) ---
# min-replicas-to-write 1
# min-replicas-max-lag 10
repl-backlog-size 256mb
repl-backlog-ttl 3600
repl-diskless-sync yes
repl-diskless-sync-delay 5

# --- CLUSTER (descomente se cluster) ---
# cluster-enabled yes
# cluster-config-file nodes.conf
# cluster-node-timeout 5000
# cluster-migration-barrier 1
# cluster-require-full-coverage yes
```

---

## <a id="referencias">Referências</a>

- Redis official: [https://redis.io/](https://redis.io/)
- Redis commands: [https://redis.io/commands](https://redis.io/commands)
- Redis Cluster specification: [https://redis.io/docs/reference/cluster-spec/](https://redis.io/docs/reference/cluster-spec/)
- Redis Sentinel: [https://redis.io/docs/management/sentinel/](https://redis.io/docs/management/sentinel/)
- Redis Streams intro: [https://redis.io/docs/data-types/streams/](https://redis.io/docs/data-types/streams/)
- Redis Functions: [https://redis.io/docs/interact/programmability/functions-intro/](https://redis.io/docs/interact/programmability/functions-intro/)
- Redis ACLs: [https://redis.io/docs/management/security/acl/](https://redis.io/docs/management/security/acl/)
- Redis Modules: [https://redis.io/modules](https://redis.io/modules)
- Redis Best Practices: [https://redis.io/docs/management/optimization/](https://redis.io/docs/management/optimization/)
- Redlock algorithm: [https://redis.io/docs/reference/patterns/distributed-locks/](https://redis.io/docs/reference/patterns/distributed-locks/)
- redis-py (Python client): [https://github.com/redis/redis-py](https://github.com/redis/redis-py)
- Redis Exporter (Prometheus): [https://github.com/oliver006/redis_exporter](https://github.com/oliver006/redis_exporter)
- Redis Stack (módulos integrados): [https://redis.io/docs/stack/](https://redis.io/docs/stack/)
- Martin Kleppmann — análise do Redlock: [https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)  
