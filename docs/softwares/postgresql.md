# PostgreSQL: Guia Completo — do Básico ao Avançado

Guia abrangente do PostgreSQL cobrindo instalação, configuração, administração, desenvolvimento SQL, performance, segurança e operações avançadas.

---

## Índice

- [1. Instalação](#instalacao)
    - [Pré-requisitos](#pre-requisitos)
    - [Debian/Ubuntu](#debian-ubuntu)
    - [RHEL/CentOS/Rocky/Alma](#rhel-centos-rocky-alma)
    - [Fedora](#fedora)
    - [Pós-instalação](#pos-instalacao)
    - [Verificação rápida](#verificacao-rapida)
- [2. Arquitetura e Conceitos Fundamentais](#arquitetura)
- [3. Configuração (postgresql.conf)](#configuracao)
- [4. Autenticação (pg_hba.conf)](#autenticacao)
- [5. O cliente psql](#psql)
- [6. Gerenciamento de Bancos e Schemas](#bancos-schemas)
- [7. Usuários, Roles e Permissões](#roles-permissoes)
- [8. Tipos de Dados](#tipos-dados)
- [9. DDL — Tabelas, Constraints e Sequences](#ddl)
- [10. DML — INSERT, UPDATE, DELETE e UPSERT](#dml)
- [11. Consultas — SELECT, JOINs, CTEs e Window Functions](#consultas)
- [12. Índices](#indices)
- [13. Views e Materialized Views](#views)
- [14. Funções e Procedures (PL/pgSQL)](#plpgsql)
- [15. Triggers](#triggers)
- [16. Transações e Controle de Concorrência (MVCC)](#transacoes)
- [17. JSON e JSONB](#jsonb)
- [18. Full-Text Search](#fts)
- [19. Particionamento de Tabelas](#particionamento)
- [20. Performance e Tuning](#performance)
- [21. Backup e Restore](#backup)
- [22. Replicação](#replicacao)
- [23. Segurança](#seguranca)
- [24. Monitoramento e Manutenção](#monitoramento)
- [25. Extensões Úteis](#extensoes)
- [26. Upgrade de Versão](#upgrade)
- [27. Referência Rápida de Comandos](#referencia)

---

## <a id="instalacao">1. Instalação</a>

### <a id="pre-requisitos">Pré-requisitos</a>

- Acesso com privilégios de administrador (`sudo`).
- Conexão com a internet para baixar pacotes.

### <a id="debian-ubuntu">Debian/Ubuntu</a>

1. Atualize o índice de pacotes:

```bash
sudo apt update
```

2. Instale o PostgreSQL e utilitários comuns:

```bash
sudo apt install -y postgresql postgresql-contrib
```

3. Verifique o serviço:

```bash
sudo systemctl status postgresql
```

4. Acesse o shell do PostgreSQL:

```bash
sudo -i -u postgres
psql
```

**Instalação de versão específica via repositório oficial (PGDG):**

```bash
sudo apt install -y gnupg2 lsb-release
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
sudo apt update
sudo apt install -y postgresql-16 postgresql-contrib-16
```

### <a id="rhel-centos-rocky-alma">RHEL/CentOS/Rocky/Alma</a>

1. Instale o repositório oficial do PostgreSQL (PGDG):

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

2. Desabilite o módulo padrão do PostgreSQL (se aplicável):

```bash
sudo dnf -qy module disable postgresql
```

3. Instale o servidor e contribuições:

```bash
sudo dnf install -y postgresql16-server postgresql16-contrib
```

4. Inicialize o banco de dados e habilite o serviço:

```bash
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable --now postgresql-16
```

5. Verifique o status:

```bash
sudo systemctl status postgresql-16
```

### <a id="fedora">Fedora</a>

1. Instale o PostgreSQL:

```bash
sudo dnf install -y postgresql-server postgresql-contrib
```

2. Inicialize o banco de dados e habilite o serviço:

```bash
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql
```

### <a id="pos-instalacao">Pós-instalação</a>

**Alterar a senha do usuário postgres:**

```bash
sudo -i -u postgres
psql
```

```sql
ALTER USER postgres WITH PASSWORD 'sua_senha_segura';
```

**Criar um banco e um usuário:**

```sql
CREATE USER app_user WITH PASSWORD 'senha_forte';
CREATE DATABASE app_db OWNER app_user;
```

**Permitir conexões remotas (opcional):**

1. Em `postgresql.conf`:

```conf
listen_addresses = '*'
```

2. Em `pg_hba.conf`:

```conf
host    all    all    192.168.1.0/24    scram-sha-256
```

3. Reinicie:

```bash
sudo systemctl restart postgresql
```

### <a id="verificacao-rapida">Verificação rápida</a>

```bash
# Versão instalada
psql --version

# Conexão local
psql -U postgres -h localhost

# Listar bancos
psql -U postgres -l
```

---

## <a id="arquitetura">2. Arquitetura e Conceitos Fundamentais</a>

### Processos principais

O PostgreSQL utiliza um modelo **multiprocesso** (não multithreaded):

| Processo | Função |
|---|---|
| `postmaster` | Processo principal; escuta conexões e faz fork de backends |
| `backend` | Um processo por conexão de cliente |
| `WAL writer` | Grava buffers do WAL em disco |
| `checkpointer` | Escreve dirty pages do shared buffer para disco |
| `autovacuum launcher` | Inicia workers de autovacuum |
| `bgwriter` | Escreve dirty buffers em background |
| `stats collector` | Coleta estatísticas de uso |
| `WAL sender/receiver` | Processos de replicação |

### Estrutura de armazenamento

```
$PGDATA/
├── base/              # Dados das databases (um subdir por OID)
├── global/            # Catálogos compartilhados (pg_database, pg_auth...)
├── pg_wal/            # Write-Ahead Logs (WAL)
├── pg_xact/           # Status de transações (commit log)
├── pg_stat/           # Estatísticas persistentes
├── pg_tblspc/         # Symlinks para tablespaces
├── postgresql.conf    # Configuração principal
├── pg_hba.conf        # Autenticação
├── pg_ident.conf      # Mapeamento de usuários do SO
└── postmaster.pid     # PID do processo principal
```

### MVCC (Multi-Version Concurrency Control)

O PostgreSQL usa MVCC para controlar concorrência sem locks de leitura:

- Cada `INSERT` cria uma nova versão da tupla (com `xmin` = ID da transação).
- Cada `UPDATE` cria uma nova versão e marca a antiga como expirada (`xmax`).
- Cada `DELETE` marca a tupla com `xmax`.
- Leitores nunca bloqueiam escritores e vice-versa.
- O `VACUUM` remove tuplas mortas (dead tuples) que não são mais visíveis por nenhuma transação.

### WAL (Write-Ahead Logging)

- Toda modificação é primeiro gravada no WAL antes de ir para os datafiles.
- Garante durabilidade (crash recovery) e é a base para replicação e PITR.
- Segmentos de 16 MB por padrão (configurável na inicialização com `--wal-segsize`).

### Tablespaces

Permitem armazenar dados em discos/partições diferentes:

```sql
CREATE TABLESPACE fast_disk LOCATION '/mnt/ssd/pg_data';
CREATE TABLE logs (...) TABLESPACE fast_disk;
```

### Catálogos do sistema

Tabelas internas com metadados:

| Catálogo | Descrição |
|---|---|
| `pg_class` | Tabelas, índices, sequences, views |
| `pg_attribute` | Colunas das relações |
| `pg_namespace` | Schemas |
| `pg_database` | Bancos de dados |
| `pg_authid` | Roles/usuários |
| `pg_index` | Informações de índices |
| `pg_stat_activity` | Sessões ativas |
| `pg_locks` | Locks ativos |

---

## <a id="configuracao">3. Configuração (postgresql.conf)</a>

Localização do arquivo:

```sql
SHOW config_file;
```

### Parâmetros essenciais

#### Memória

```conf
# Memória compartilhada para cache (25% da RAM é um bom ponto de partida)
shared_buffers = '4GB'

# Memória por operação de sort/hash (por query, não global)
work_mem = '64MB'

# Memória para operações de manutenção (VACUUM, CREATE INDEX)
maintenance_work_mem = '512MB'

# Estimativa de cache do SO disponível (50-75% da RAM)
effective_cache_size = '12GB'

# Memória para WAL buffers
wal_buffers = '64MB'
```

#### Conexões

```conf
max_connections = 200
superuser_reserved_connections = 3
```

#### WAL e Checkpoint

```conf
wal_level = replica                    # minimal, replica ou logical
max_wal_size = '2GB'                   # Tamanho máximo antes de forçar checkpoint
min_wal_size = '512MB'
checkpoint_completion_target = 0.9     # Espalha I/O do checkpoint
checkpoint_timeout = '10min'
```

#### Autovacuum

```conf
autovacuum = on
autovacuum_max_workers = 3
autovacuum_vacuum_scale_factor = 0.1     # Vacuum quando 10% das tuplas forem mortas
autovacuum_analyze_scale_factor = 0.05   # Analyze quando 5% mudaram
autovacuum_vacuum_cost_delay = '2ms'     # Throttle para reduzir impacto em I/O
```

#### Logging

```conf
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = '1d'
log_rotation_size = '100MB'
log_min_duration_statement = '500ms'   # Logar queries acima de 500ms
log_statement = 'ddl'                  # none, ddl, mod, all
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0                     # Logar toda criação de arquivo temp
```

#### Locale e Timezone

```conf
timezone = 'America/Sao_Paulo'
lc_messages = 'pt_BR.UTF-8'
```

### Aplicando alterações

```sql
-- Recarregar sem reiniciar (funciona para a maioria dos parâmetros)
SELECT pg_reload_conf();

-- Ou via linha de comando
-- sudo systemctl reload postgresql

-- Verificar se um parâmetro exige restart
SELECT name, setting, context FROM pg_settings WHERE name = 'shared_buffers';
-- context = 'postmaster' → exige restart
-- context = 'sighup'     → basta reload
-- context = 'user'       → alterável por sessão
```

### Alterando parâmetros por sessão/role/banco

```sql
-- Por sessão
SET work_mem = '128MB';

-- Por role
ALTER ROLE app_user SET work_mem = '128MB';

-- Por banco
ALTER DATABASE app_db SET work_mem = '128MB';
```

---

## <a id="autenticacao">4. Autenticação (pg_hba.conf)</a>

Localização:

```sql
SHOW hba_file;
```

### Formato das linhas

```
TIPO    BANCO       USUÁRIO     ENDEREÇO         MÉTODO
```

### Tipos de conexão

| Tipo | Descrição |
|---|---|
| `local` | Conexão via Unix socket |
| `host` | Conexão TCP/IP (com ou sem SSL) |
| `hostssl` | Apenas conexões SSL |
| `hostnossl` | Apenas conexões sem SSL |
| `hostgssenc` | Apenas com GSSAPI encryption |

### Métodos de autenticação

| Método | Descrição |
|---|---|
| `trust` | Sem senha (perigoso em produção) |
| `reject` | Rejeita a conexão |
| `scram-sha-256` | Hash seguro (recomendado) |
| `md5` | Hash MD5 (legado, menos seguro) |
| `password` | Senha em texto claro (evitar) |
| `peer` | Usa o usuário do SO (apenas `local`) |
| `ident` | Mapeamento via ident server |
| `cert` | Autenticação por certificado SSL do cliente |
| `gss` | Autenticação Kerberos/GSSAPI |
| `ldap` | Autenticação via servidor LDAP |
| `pam` | Autenticação via PAM |

### Exemplos práticos

```conf
# Conexão local via peer (padrão no Debian/Ubuntu)
local   all    postgres                        peer

# Senha para conexões locais
local   all    all                             scram-sha-256

# Rede local via senha
host    all    all    192.168.1.0/24           scram-sha-256

# SSL obrigatório para rede externa
hostssl all    all    10.0.0.0/8               scram-sha-256

# Replicação
host    replication   replicator   10.0.0.2/32   scram-sha-256

# Rejeitar tudo mais
host    all    all    0.0.0.0/0                reject
```

As regras são avaliadas de cima para baixo; a primeira que casar é aplicada.

Após alterar, recarregue:

```bash
sudo systemctl reload postgresql
```

---

## <a id="psql">5. O cliente psql</a>

### Conexão

```bash
# Conexão local
psql -U postgres

# Conexão remota
psql -h 192.168.1.10 -p 5432 -U app_user -d app_db

# Via connection string
psql "postgresql://app_user:senha@host:5432/app_db?sslmode=require"

# Variáveis de ambiente
export PGHOST=localhost PGPORT=5432 PGUSER=app_user PGDATABASE=app_db
psql
```

### Meta-comandos essenciais

| Comando | Descrição |
|---|---|
| `\l` | Listar bancos de dados |
| `\c nome_db` | Conectar a outro banco |
| `\dt` | Listar tabelas do schema atual |
| `\dt+` | Listar tabelas com tamanho |
| `\dt schema.*` | Listar tabelas de um schema |
| `\d tabela` | Descrever tabela (colunas, tipos, constraints) |
| `\di` | Listar índices |
| `\dv` | Listar views |
| `\df` | Listar funções |
| `\du` | Listar roles/usuários |
| `\dn` | Listar schemas |
| `\dx` | Listar extensões instaladas |
| `\ds` | Listar sequences |
| `\dp` | Listar permissões (ACL) |
| `\timing` | Ativar/desativar exibição de tempo de execução |
| `\x` | Ativar/desativar exibição expandida |
| `\e` | Abrir editor externo |
| `\i arquivo.sql` | Executar arquivo SQL |
| `\o arquivo.txt` | Redirecionar saída para arquivo |
| `\copy` | Copiar dados de/para arquivo CSV |
| `\watch 5` | Repetir última query a cada 5 segundos |
| `\q` | Sair do psql |
| `\?` | Ajuda dos meta-comandos |
| `\h COMANDO` | Ajuda de um comando SQL |

### Arquivo .psqlrc

Personalizações carregadas automaticamente (`~/.psqlrc`):

```sql
\set PROMPT1 '%n@%/%R%# '
\set PROMPT2 '%R%# '
\timing on
\pset null '(null)'
\set HISTSIZE 10000
\set COMP_KEYWORD_CASE upper
```

### Importação/Exportação CSV

```sql
-- Exportar
\copy (SELECT * FROM clientes WHERE ativo) TO '/tmp/clientes.csv' WITH CSV HEADER

-- Importar
\copy clientes(nome, email) FROM '/tmp/novos.csv' WITH CSV HEADER
```

---

## <a id="bancos-schemas">6. Gerenciamento de Bancos e Schemas</a>

### Bancos de dados

```sql
-- Criar banco
CREATE DATABASE meu_app
    OWNER = app_user
    ENCODING = 'UTF8'
    LC_COLLATE = 'pt_BR.UTF-8'
    LC_CTYPE = 'pt_BR.UTF-8'
    TEMPLATE = template0
    CONNECTION LIMIT = 100;

-- Criar banco a partir de outro (cópia)
CREATE DATABASE meu_app_dev TEMPLATE meu_app;

-- Renomear
ALTER DATABASE meu_app RENAME TO meu_app_v2;

-- Alterar owner
ALTER DATABASE meu_app OWNER TO novo_owner;

-- Definir parâmetros por banco
ALTER DATABASE meu_app SET timezone = 'America/Sao_Paulo';

-- Remover
DROP DATABASE IF EXISTS meu_app_dev;

-- Tamanho de cada banco
SELECT datname, pg_size_pretty(pg_database_size(datname)) AS tamanho
FROM pg_database ORDER BY pg_database_size(datname) DESC;
```

### Schemas

Schemas organizam objetos dentro de um banco de dados:

```sql
-- Criar schema
CREATE SCHEMA vendas AUTHORIZATION app_user;

-- Criar tabela dentro de um schema
CREATE TABLE vendas.pedidos (
    id SERIAL PRIMARY KEY,
    valor NUMERIC(10,2)
);

-- Definir o search_path (quais schemas são visíveis por padrão)
SET search_path TO vendas, public;

-- Search_path permanente para uma role
ALTER ROLE app_user SET search_path TO vendas, public;

-- Remover schema (CASCADE remove todos os objetos dentro)
DROP SCHEMA vendas CASCADE;

-- Listar schemas
SELECT schema_name FROM information_schema.schemata;
```

---

## <a id="roles-permissoes">7. Usuários, Roles e Permissões</a>

No PostgreSQL, **usuários** e **grupos** são ambos **roles**. Um role com `LOGIN` é um "usuário".

### Criação e gerenciamento de roles

```sql
-- Criar role de login (usuário)
CREATE ROLE app_user WITH
    LOGIN
    PASSWORD 'senha_forte'
    VALID UNTIL '2027-01-01'
    CONNECTION LIMIT 50;

-- Criar role de grupo (sem login)
CREATE ROLE equipe_backend NOLOGIN;

-- Adicionar usuário ao grupo
GRANT equipe_backend TO app_user;

-- Criar superuser (com cuidado)
CREATE ROLE admin_master WITH SUPERUSER LOGIN PASSWORD 'ultra_segura';

-- Alterar atributos
ALTER ROLE app_user CREATEDB;
ALTER ROLE app_user SET statement_timeout = '30s';

-- Remover role
DROP ROLE IF EXISTS app_user;
```

### Atributos de roles

| Atributo | Descrição |
|---|---|
| `SUPERUSER` | Ignora todas as verificações de permissão |
| `CREATEDB` | Pode criar bancos |
| `CREATEROLE` | Pode criar e gerenciar roles |
| `LOGIN` | Pode conectar ao servidor |
| `REPLICATION` | Pode iniciar streaming replication |
| `BYPASSRLS` | Ignora Row Level Security |
| `INHERIT` | Herda privilégios de roles membras |
| `CONNECTION LIMIT n` | Limite de conexões simultâneas |
| `VALID UNTIL 'data'` | Expiração da senha |

### Privilégios (GRANT/REVOKE)

```sql
-- Conceder acesso ao banco
GRANT CONNECT ON DATABASE app_db TO app_user;

-- Conceder uso do schema
GRANT USAGE ON SCHEMA vendas TO app_user;

-- Conceder em tabelas
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA vendas TO app_user;

-- Conceder em tabelas futuras
ALTER DEFAULT PRIVILEGES IN SCHEMA vendas
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;

-- Conceder em sequences
GRANT USAGE ON ALL SEQUENCES IN SCHEMA vendas TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA vendas GRANT USAGE ON SEQUENCES TO app_user;

-- Conceder execução de funções
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA vendas TO app_user;

-- Conceder todos os privilégios
GRANT ALL PRIVILEGES ON DATABASE app_db TO app_user;

-- Revogar
REVOKE DELETE ON vendas.pedidos FROM app_user;

-- Tornar role OWNER de um schema
ALTER SCHEMA vendas OWNER TO equipe_backend;
```

### Verificar permissões

```sql
-- Permissões de tabelas
SELECT grantee, table_schema, table_name, privilege_type
FROM information_schema.table_privileges
WHERE table_schema = 'vendas';

-- Permissões no psql
\dp vendas.*
```

---

## <a id="tipos-dados">8. Tipos de Dados</a>

### Numéricos

| Tipo | Tamanho | Faixa |
|---|---|---|
| `SMALLINT` | 2 bytes | -32.768 a 32.767 |
| `INTEGER` | 4 bytes | -2.147.483.648 a 2.147.483.647 |
| `BIGINT` | 8 bytes | -9.2×10¹⁸ a 9.2×10¹⁸ |
| `NUMERIC(p,s)` | variável | Precisão exata (financeiro) |
| `REAL` | 4 bytes | 6 dígitos de precisão |
| `DOUBLE PRECISION` | 8 bytes | 15 dígitos de precisão |
| `SERIAL` | 4 bytes | Auto-incremento (1 a 2.147.483.647) |
| `BIGSERIAL` | 8 bytes | Auto-incremento grande |

### Texto

| Tipo | Descrição |
|---|---|
| `VARCHAR(n)` | Texto com limite de `n` caracteres |
| `CHAR(n)` | Texto de tamanho fixo (preenchido com espaços) |
| `TEXT` | Texto sem limite de tamanho |

> `TEXT` e `VARCHAR` sem limite têm a mesma performance no PostgreSQL.

### Data e hora

| Tipo | Descrição |
|---|---|
| `DATE` | Data sem hora |
| `TIME` | Hora sem data |
| `TIMESTAMP` | Data e hora sem timezone |
| `TIMESTAMPTZ` | Data e hora com timezone (recomendado) |
| `INTERVAL` | Duração de tempo |

```sql
SELECT NOW();                                      -- timestamp com tz atual
SELECT CURRENT_DATE;                               -- data atual
SELECT CURRENT_DATE + INTERVAL '30 days';          -- daqui a 30 dias
SELECT age(TIMESTAMP '2000-01-01', CURRENT_DATE);  -- diferença
```

### Boolean

```sql
-- Aceita: TRUE, FALSE, 't', 'f', 'yes', 'no', '1', '0', NULL
CREATE TABLE config (ativo BOOLEAN DEFAULT TRUE);
```

### UUID

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE sessoes (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    usuario_id INTEGER
);

-- Ou com gen_random_uuid() (nativo desde PostgreSQL 13):
CREATE TABLE sessoes (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY
);
```

### Arrays

```sql
CREATE TABLE produtos (
    nome TEXT,
    tags TEXT[]
);

INSERT INTO produtos VALUES ('Notebook', ARRAY['eletrônico', 'informática']);
INSERT INTO produtos VALUES ('Caderno', '{"papelaria","escola"}');

-- Consultar
SELECT * FROM produtos WHERE 'eletrônico' = ANY(tags);
SELECT * FROM produtos WHERE tags @> ARRAY['informática'];

-- Funções úteis
SELECT array_length(tags, 1) FROM produtos;
SELECT unnest(tags) FROM produtos;
```

### JSON e JSONB

Veja seção [JSON e JSONB](#jsonb).

### Tipos especiais

| Tipo | Descrição |
|---|---|
| `BYTEA` | Dados binários |
| `INET` / `CIDR` | Endereços IP/redes |
| `MACADDR` | Endereços MAC |
| `MONEY` | Valores monetários (prefer `NUMERIC`) |
| `HSTORE` | Pares chave-valor (extensão) |
| `TSVECTOR` / `TSQUERY` | Full-text search |
| `POINT`, `LINE`, `BOX`, `CIRCLE` | Tipos geométricos |
| `DATERANGE`, `INT4RANGE`, `TSTZRANGE` | Tipos de intervalo (range) |

### Tipos enumerados

```sql
CREATE TYPE status_pedido AS ENUM ('pendente', 'processando', 'entregue', 'cancelado');

CREATE TABLE pedidos (
    id SERIAL PRIMARY KEY,
    status status_pedido DEFAULT 'pendente'
);

-- Adicionar valor ao enum
ALTER TYPE status_pedido ADD VALUE 'devolvido' AFTER 'entregue';
```

### Tipos compostos

```sql
CREATE TYPE endereco AS (
    rua TEXT,
    cidade TEXT,
    estado CHAR(2),
    cep VARCHAR(9)
);

CREATE TABLE clientes (
    id SERIAL PRIMARY KEY,
    nome TEXT,
    end_residencial endereco
);

INSERT INTO clientes (nome, end_residencial)
VALUES ('Ana', ROW('Rua X, 100', 'São Paulo', 'SP', '01000-000'));

SELECT (end_residencial).cidade FROM clientes;
```

### Domínios (tipos customizados com constraints)

```sql
CREATE DOMAIN email AS TEXT
    CHECK (VALUE ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$');

CREATE TABLE usuarios (
    id SERIAL PRIMARY KEY,
    email email NOT NULL
);
```

---

## <a id="ddl">9. DDL — Tabelas, Constraints e Sequences</a>

### Criar tabelas

```sql
CREATE TABLE clientes (
    id          BIGSERIAL PRIMARY KEY,
    nome        TEXT NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    cpf         CHAR(11) UNIQUE,
    nascimento  DATE,
    saldo       NUMERIC(12,2) DEFAULT 0.00,
    ativo       BOOLEAN DEFAULT TRUE,
    criado_em   TIMESTAMPTZ DEFAULT NOW(),
    atualizado_em TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE pedidos (
    id          BIGSERIAL PRIMARY KEY,
    cliente_id  BIGINT NOT NULL REFERENCES clientes(id) ON DELETE CASCADE,
    valor_total NUMERIC(12,2) NOT NULL CHECK (valor_total >= 0),
    status      VARCHAR(20) DEFAULT 'pendente',
    criado_em   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE itens_pedido (
    id         BIGSERIAL PRIMARY KEY,
    pedido_id  BIGINT NOT NULL REFERENCES pedidos(id) ON DELETE CASCADE,
    produto    TEXT NOT NULL,
    quantidade INTEGER NOT NULL CHECK (quantidade > 0),
    preco_unit NUMERIC(10,2) NOT NULL,
    UNIQUE (pedido_id, produto)
);
```

### Constraints

```sql
-- Primary Key
ALTER TABLE tabela ADD PRIMARY KEY (id);

-- Unique
ALTER TABLE clientes ADD CONSTRAINT uk_email UNIQUE (email);

-- Foreign Key
ALTER TABLE pedidos ADD CONSTRAINT fk_cliente
    FOREIGN KEY (cliente_id) REFERENCES clientes(id)
    ON DELETE CASCADE
    ON UPDATE CASCADE;

-- Check
ALTER TABLE pedidos ADD CONSTRAINT chk_valor CHECK (valor_total >= 0);

-- Not Null
ALTER TABLE clientes ALTER COLUMN nome SET NOT NULL;

-- Exclusion Constraint (evitar sobreposição de intervalos)
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE TABLE reservas (
    sala INT,
    periodo TSTZRANGE,
    EXCLUDE USING GIST (sala WITH =, periodo WITH &&)
);
```

### Alterar tabelas

```sql
-- Adicionar coluna
ALTER TABLE clientes ADD COLUMN telefone VARCHAR(20);

-- Remover coluna
ALTER TABLE clientes DROP COLUMN telefone;

-- Renomear coluna
ALTER TABLE clientes RENAME COLUMN nome TO nome_completo;

-- Alterar tipo
ALTER TABLE clientes ALTER COLUMN email TYPE TEXT;

-- Definir/remover default
ALTER TABLE clientes ALTER COLUMN ativo SET DEFAULT TRUE;
ALTER TABLE clientes ALTER COLUMN ativo DROP DEFAULT;

-- Renomear tabela
ALTER TABLE clientes RENAME TO customers;
```

### Sequences

```sql
-- Criar sequence manualmente
CREATE SEQUENCE pedido_seq START 1000 INCREMENT 1;

-- Usar
INSERT INTO pedidos (id, ...) VALUES (nextval('pedido_seq'), ...);

-- Ver valor atual
SELECT currval('pedido_seq');

-- Resetar
ALTER SEQUENCE pedido_seq RESTART WITH 1;

-- Identity columns (padrão SQL, preferido sobre SERIAL)
CREATE TABLE produtos (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome TEXT NOT NULL
);
```

### Tabelas temporárias

```sql
-- Existem apenas durante a sessão
CREATE TEMP TABLE tmp_importacao (
    linha INT,
    dados TEXT
);

-- Ou removida ao final da transação
CREATE TEMP TABLE tmp_calc (...) ON COMMIT DROP;
```

### Tabelas UNLOGGED

Sem WAL — mais rápidas para escrita, mas **não sobrevivem a crash**:

```sql
CREATE UNLOGGED TABLE cache_sessoes (
    token TEXT PRIMARY KEY,
    dados JSONB,
    expira_em TIMESTAMPTZ
);
```

---

## <a id="dml">10. DML — INSERT, UPDATE, DELETE e UPSERT</a>

### INSERT

```sql
-- Inserção simples
INSERT INTO clientes (nome, email) VALUES ('Ana Silva', 'ana@email.com');

-- Múltiplas linhas
INSERT INTO clientes (nome, email) VALUES
    ('Carlos', 'carlos@email.com'),
    ('Maria', 'maria@email.com'),
    ('João', 'joao@email.com');

-- INSERT com RETURNING (retorna dados inseridos)
INSERT INTO clientes (nome, email)
VALUES ('Pedro', 'pedro@email.com')
RETURNING id, criado_em;

-- INSERT a partir de SELECT
INSERT INTO clientes_backup (nome, email)
SELECT nome, email FROM clientes WHERE ativo = false;

-- INSERT com CTE
WITH novos AS (
    SELECT * FROM staging_clientes WHERE validado = true
)
INSERT INTO clientes (nome, email)
SELECT nome, email FROM novos;
```

### UPDATE

```sql
-- Update simples
UPDATE clientes SET ativo = false WHERE ultimo_login < NOW() - INTERVAL '1 year';

-- Update múltiplas colunas
UPDATE clientes
SET nome = 'Ana Maria', atualizado_em = NOW()
WHERE id = 1;

-- Update com RETURNING
UPDATE clientes SET ativo = false WHERE id = 5 RETURNING *;

-- Update com JOIN (FROM)
UPDATE pedidos p
SET status = 'cancelado'
FROM clientes c
WHERE p.cliente_id = c.id AND c.ativo = false;

-- Update com subquery
UPDATE produtos
SET preco = preco * 1.10
WHERE categoria_id IN (SELECT id FROM categorias WHERE nome = 'importados');
```

### DELETE

```sql
-- Delete simples
DELETE FROM clientes WHERE ativo = false AND ultimo_login < '2024-01-01';

-- Delete com RETURNING
DELETE FROM sessoes WHERE expira_em < NOW() RETURNING id, usuario_id;

-- Delete com JOIN (USING)
DELETE FROM itens_pedido i
USING pedidos p
WHERE i.pedido_id = p.id AND p.status = 'cancelado';

-- Remover todas as linhas (mais rápido que DELETE sem WHERE)
TRUNCATE TABLE logs_temp;
TRUNCATE TABLE pedidos, itens_pedido CASCADE;
```

### UPSERT (INSERT ... ON CONFLICT)

```sql
-- Inserir ou atualizar se email já existir
INSERT INTO clientes (nome, email, telefone)
VALUES ('Ana', 'ana@email.com', '11999999999')
ON CONFLICT (email)
DO UPDATE SET
    nome = EXCLUDED.nome,
    telefone = EXCLUDED.telefone,
    atualizado_em = NOW();

-- Inserir ou ignorar duplicatas
INSERT INTO clientes (nome, email)
VALUES ('Ana', 'ana@email.com')
ON CONFLICT (email) DO NOTHING;

-- Upsert com RETURNING
INSERT INTO config (chave, valor)
VALUES ('max_tentativas', '5')
ON CONFLICT (chave)
DO UPDATE SET valor = EXCLUDED.valor
RETURNING *;
```

### COPY (carga em massa)

```sql
-- Importar CSV (comando do servidor — requer acesso ao filesystem do servidor)
COPY clientes(nome, email) FROM '/tmp/clientes.csv' WITH (FORMAT csv, HEADER true);

-- Exportar CSV
COPY (SELECT * FROM clientes WHERE ativo) TO '/tmp/ativos.csv' WITH (FORMAT csv, HEADER true);

-- Via psql (\copy roda no cliente)
\copy clientes(nome, email) FROM 'clientes.csv' WITH CSV HEADER
```

---

## <a id="consultas">11. Consultas — SELECT, JOINs, CTEs e Window Functions</a>

### SELECT básico

```sql
SELECT nome, email FROM clientes WHERE ativo = true ORDER BY nome LIMIT 10;

-- Alias
SELECT c.nome AS cliente, p.valor_total AS valor
FROM clientes c
JOIN pedidos p ON p.cliente_id = c.id;

-- DISTINCT
SELECT DISTINCT cidade FROM clientes;
SELECT DISTINCT ON (cidade) cidade, nome FROM clientes ORDER BY cidade, nome;
```

### Cláusula WHERE — operadores

```sql
-- Comparação
WHERE idade >= 18 AND idade <= 65
WHERE idade BETWEEN 18 AND 65

-- Texto
WHERE nome LIKE 'Ana%'              -- começa com Ana
WHERE nome ILIKE '%silva%'           -- case-insensitive
WHERE nome SIMILAR TO '%(Silva|Santos)%'

-- Regex POSIX
WHERE email ~ '^[a-z]+@empresa\.com$'
WHERE email ~* 'gmail'              -- case-insensitive

-- NULL
WHERE telefone IS NULL
WHERE telefone IS NOT NULL

-- Listas
WHERE status IN ('ativo', 'pendente')
WHERE id = ANY(ARRAY[1, 2, 3])

-- Existência
WHERE EXISTS (SELECT 1 FROM pedidos WHERE cliente_id = clientes.id)
```

### JOINs

```sql
-- INNER JOIN
SELECT c.nome, p.valor_total
FROM clientes c
INNER JOIN pedidos p ON p.cliente_id = c.id;

-- LEFT JOIN (todos os clientes, com ou sem pedidos)
SELECT c.nome, COALESCE(SUM(p.valor_total), 0) AS total
FROM clientes c
LEFT JOIN pedidos p ON p.cliente_id = c.id
GROUP BY c.nome;

-- RIGHT JOIN
SELECT p.id, c.nome
FROM pedidos p
RIGHT JOIN clientes c ON c.id = p.cliente_id;

-- FULL OUTER JOIN
SELECT * FROM tabela_a a
FULL OUTER JOIN tabela_b b ON a.chave = b.chave;

-- CROSS JOIN (produto cartesiano)
SELECT * FROM cores CROSS JOIN tamanhos;

-- LATERAL JOIN (subquery correlacionada como tabela)
SELECT c.nome, ult.valor_total, ult.criado_em
FROM clientes c
CROSS JOIN LATERAL (
    SELECT valor_total, criado_em
    FROM pedidos
    WHERE cliente_id = c.id
    ORDER BY criado_em DESC
    LIMIT 3
) ult;

-- Self JOIN
SELECT e.nome AS funcionario, g.nome AS gerente
FROM funcionarios e
LEFT JOIN funcionarios g ON e.gerente_id = g.id;
```

### Agregações

```sql
SELECT
    cidade,
    COUNT(*) AS total_clientes,
    AVG(saldo) AS saldo_medio,
    SUM(saldo) AS saldo_total,
    MIN(saldo) AS menor_saldo,
    MAX(saldo) AS maior_saldo
FROM clientes
GROUP BY cidade
HAVING COUNT(*) >= 5
ORDER BY total_clientes DESC;

-- Agregação com FILTER
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE ativo) AS ativos,
    COUNT(*) FILTER (WHERE NOT ativo) AS inativos
FROM clientes;

-- GROUPING SETS / ROLLUP / CUBE
SELECT cidade, estado, COUNT(*)
FROM clientes
GROUP BY GROUPING SETS ((cidade, estado), (estado), ());

SELECT estado, cidade, COUNT(*)
FROM clientes
GROUP BY ROLLUP (estado, cidade);
```

### CTEs (Common Table Expressions)

```sql
-- CTE simples
WITH clientes_vip AS (
    SELECT * FROM clientes WHERE saldo > 10000
)
SELECT cv.nome, COUNT(p.id) AS total_pedidos
FROM clientes_vip cv
LEFT JOIN pedidos p ON p.cliente_id = cv.id
GROUP BY cv.nome;

-- Múltiplas CTEs
WITH
    vendas_mes AS (
        SELECT cliente_id, SUM(valor_total) AS total
        FROM pedidos
        WHERE criado_em >= date_trunc('month', CURRENT_DATE)
        GROUP BY cliente_id
    ),
    top_clientes AS (
        SELECT cliente_id, total
        FROM vendas_mes
        ORDER BY total DESC
        LIMIT 10
    )
SELECT c.nome, t.total
FROM top_clientes t
JOIN clientes c ON c.id = t.cliente_id;

-- CTE recursiva (hierarquia)
WITH RECURSIVE hierarquia AS (
    -- Caso base: raiz
    SELECT id, nome, gerente_id, 1 AS nivel
    FROM funcionarios
    WHERE gerente_id IS NULL

    UNION ALL

    -- Caso recursivo
    SELECT f.id, f.nome, f.gerente_id, h.nivel + 1
    FROM funcionarios f
    JOIN hierarquia h ON f.gerente_id = h.id
)
SELECT * FROM hierarquia ORDER BY nivel, nome;

-- CTE com INSERT/UPDATE/DELETE (writeable CTEs)
WITH removidos AS (
    DELETE FROM sessoes WHERE expira_em < NOW() RETURNING *
)
INSERT INTO sessoes_expiradas SELECT * FROM removidos;
```

### Window Functions

```sql
-- ROW_NUMBER, RANK, DENSE_RANK
SELECT
    nome,
    cidade,
    saldo,
    ROW_NUMBER() OVER (ORDER BY saldo DESC) AS posicao,
    RANK() OVER (ORDER BY saldo DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY saldo DESC) AS dense_rank
FROM clientes;

-- Particionamento
SELECT
    cidade,
    nome,
    saldo,
    ROW_NUMBER() OVER (PARTITION BY cidade ORDER BY saldo DESC) AS rank_na_cidade,
    SUM(saldo) OVER (PARTITION BY cidade) AS saldo_total_cidade,
    AVG(saldo) OVER (PARTITION BY cidade) AS saldo_medio_cidade
FROM clientes;

-- LAG / LEAD (valor anterior/próximo)
SELECT
    data,
    receita,
    LAG(receita) OVER (ORDER BY data) AS receita_anterior,
    receita - LAG(receita) OVER (ORDER BY data) AS variacao
FROM vendas_diarias;

-- Running total / Moving average
SELECT
    data,
    receita,
    SUM(receita) OVER (ORDER BY data) AS acumulado,
    AVG(receita) OVER (ORDER BY data ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS media_movel_7d
FROM vendas_diarias;

-- FIRST_VALUE / LAST_VALUE / NTH_VALUE
SELECT
    nome,
    saldo,
    FIRST_VALUE(nome) OVER (ORDER BY saldo DESC) AS mais_rico,
    NTILE(4) OVER (ORDER BY saldo) AS quartil
FROM clientes;

-- Named window
SELECT
    nome,
    saldo,
    ROW_NUMBER() OVER w AS pos,
    SUM(saldo) OVER w AS acumulado
FROM clientes
WINDOW w AS (ORDER BY saldo DESC);
```

### Subqueries

```sql
-- Subquery escalar
SELECT nome, (SELECT COUNT(*) FROM pedidos WHERE cliente_id = c.id) AS total_pedidos
FROM clientes c;

-- Subquery em FROM
SELECT avg_pedidos.media
FROM (
    SELECT AVG(valor_total) AS media FROM pedidos
) avg_pedidos;

-- IN com subquery
SELECT * FROM clientes
WHERE id IN (SELECT DISTINCT cliente_id FROM pedidos WHERE valor_total > 1000);

-- EXISTS
SELECT * FROM clientes c
WHERE NOT EXISTS (SELECT 1 FROM pedidos p WHERE p.cliente_id = c.id);
```

### UNION, INTERSECT, EXCEPT

```sql
-- Union (remove duplicatas)
SELECT nome, email FROM clientes_br
UNION
SELECT nome, email FROM clientes_us;

-- Union All (mantém duplicatas)
SELECT nome FROM clientes_br
UNION ALL
SELECT nome FROM clientes_us;

-- Intersect (presentes em ambas)
SELECT email FROM clientes
INTERSECT
SELECT email FROM newsletter;

-- Except (presentes na primeira mas não na segunda)
SELECT email FROM clientes
EXCEPT
SELECT email FROM blacklist;
```

---

## <a id="indices">12. Índices</a>

### Tipos de índices

| Tipo | Uso |
|---|---|
| **B-tree** (padrão) | Comparações `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE 'abc%'` |
| **Hash** | Apenas igualdade `=` (raramente usado) |
| **GIN** | Arrays, JSONB, full-text search, trigramas |
| **GiST** | Dados espaciais, ranges, full-text search |
| **SP-GiST** | Dados particionáveis (pontos, redes IP) |
| **BRIN** | Colunas com correlação física (timestamps em append-only) |

### Criação de índices

```sql
-- B-tree simples
CREATE INDEX idx_clientes_email ON clientes (email);

-- Índice único
CREATE UNIQUE INDEX idx_clientes_cpf ON clientes (cpf);

-- Índice composto (ordem importa)
CREATE INDEX idx_pedidos_cliente_data ON pedidos (cliente_id, criado_em DESC);

-- Índice parcial (apenas linhas que atendem a condição)
CREATE INDEX idx_clientes_ativos ON clientes (email) WHERE ativo = true;

-- Índice de expressão
CREATE INDEX idx_clientes_email_lower ON clientes (lower(email));

-- Índice BRIN (eficiente para tabelas append-only)
CREATE INDEX idx_logs_data ON logs USING brin (criado_em);

-- Índice GIN para JSONB
CREATE INDEX idx_eventos_dados ON eventos USING gin (dados jsonb_path_ops);

-- Índice GIN para full-text
CREATE INDEX idx_artigos_busca ON artigos USING gin (to_tsvector('portuguese', titulo || ' ' || conteudo));

-- Índice GIN para trigramas (buscas LIKE '%termo%')
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_clientes_nome_trgm ON clientes USING gin (nome gin_trgm_ops);

-- Covering index (inclui colunas extras para index-only scan)
CREATE INDEX idx_pedidos_status ON pedidos (status) INCLUDE (valor_total, criado_em);

-- Criação concorrente (não bloqueia escrita)
CREATE INDEX CONCURRENTLY idx_pedidos_valor ON pedidos (valor_total);
```

### Fillfactor

```sql
-- Menos preenchido = mais espaço para HOT updates
CREATE INDEX idx_clientes_nome ON clientes (nome) WITH (fillfactor = 70);
```

### Remover e reindexar

```sql
DROP INDEX IF EXISTS idx_clientes_email;
DROP INDEX CONCURRENTLY idx_clientes_email;

-- Reindexar (reconstroi o índice)
REINDEX INDEX idx_clientes_nome;
REINDEX TABLE clientes;
REINDEX DATABASE meu_app;
REINDEX (VERBOSE) TABLE clientes;
```

### Verificar uso de índices

```sql
-- Índices não utilizados
SELECT
    schemaname, relname, indexrelname,
    idx_scan, idx_tup_read, idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS tamanho
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Ver se uma query usa índice
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM clientes WHERE email = 'ana@email.com';
```

---

## <a id="views">13. Views e Materialized Views</a>

### Views

```sql
-- Criar view
CREATE VIEW vw_clientes_ativos AS
SELECT id, nome, email, saldo
FROM clientes
WHERE ativo = true;

-- Consultar como tabela
SELECT * FROM vw_clientes_ativos WHERE saldo > 1000;

-- View atualizável (automaticamente se for SELECT simples de uma tabela)
UPDATE vw_clientes_ativos SET saldo = 5000 WHERE id = 1;

-- Com CHECK OPTION (impede inserir/alterar linhas que não satisfazem a view)
CREATE VIEW vw_clientes_sp AS
SELECT * FROM clientes WHERE estado = 'SP'
WITH CHECK OPTION;

-- Substituir view
CREATE OR REPLACE VIEW vw_clientes_ativos AS
SELECT id, nome, email, saldo, criado_em
FROM clientes
WHERE ativo = true;

-- Remover
DROP VIEW IF EXISTS vw_clientes_ativos;
```

### Materialized Views

Armazenam o resultado em disco — ideais para consultas pesadas:

```sql
-- Criar
CREATE MATERIALIZED VIEW mv_vendas_mensal AS
SELECT
    date_trunc('month', criado_em) AS mes,
    COUNT(*) AS total_pedidos,
    SUM(valor_total) AS receita
FROM pedidos
GROUP BY 1
ORDER BY 1;

-- Consultar (muito rápida)
SELECT * FROM mv_vendas_mensal;

-- Atualizar (refresh completo)
REFRESH MATERIALIZED VIEW mv_vendas_mensal;

-- Refresh concorrente (requer índice UNIQUE)
CREATE UNIQUE INDEX idx_mv_vendas_mes ON mv_vendas_mensal (mes);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_vendas_mensal;

-- Remover
DROP MATERIALIZED VIEW IF EXISTS mv_vendas_mensal;
```

---

## <a id="plpgsql">14. Funções e Procedures (PL/pgSQL)</a>

### Funções

```sql
-- Função simples (retorna valor)
CREATE OR REPLACE FUNCTION calcular_desconto(valor NUMERIC, percentual NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN valor * (1 - percentual / 100);
END;
$$;

SELECT calcular_desconto(100, 15);  -- 85.00

-- Função que retorna tabela
CREATE OR REPLACE FUNCTION clientes_por_cidade(p_cidade TEXT)
RETURNS TABLE (id BIGINT, nome TEXT, email TEXT)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT c.id, c.nome, c.email
    FROM clientes c
    WHERE c.cidade = p_cidade;
END;
$$;

SELECT * FROM clientes_por_cidade('São Paulo');

-- Função com variáveis e controle de fluxo
CREATE OR REPLACE FUNCTION classificar_cliente(p_id BIGINT)
RETURNS TEXT
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_saldo NUMERIC;
    v_total_pedidos INT;
    v_classificacao TEXT;
BEGIN
    SELECT saldo INTO v_saldo FROM clientes WHERE id = p_id;
    SELECT COUNT(*) INTO v_total_pedidos FROM pedidos WHERE cliente_id = p_id;

    IF v_saldo IS NULL THEN
        RAISE EXCEPTION 'Cliente % não encontrado', p_id;
    END IF;

    IF v_saldo > 50000 AND v_total_pedidos > 100 THEN
        v_classificacao := 'VIP';
    ELSIF v_saldo > 10000 OR v_total_pedidos > 50 THEN
        v_classificacao := 'Premium';
    ELSE
        v_classificacao := 'Regular';
    END IF;

    RETURN v_classificacao;
END;
$$;

-- Função com loop
CREATE OR REPLACE FUNCTION gerar_relatorio_mensal(p_ano INT)
RETURNS TABLE (mes INT, total NUMERIC)
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_mes INT;
BEGIN
    FOR v_mes IN 1..12 LOOP
        mes := v_mes;
        SELECT COALESCE(SUM(valor_total), 0) INTO total
        FROM pedidos
        WHERE EXTRACT(YEAR FROM criado_em) = p_ano
          AND EXTRACT(MONTH FROM criado_em) = v_mes;
        RETURN NEXT;
    END LOOP;
END;
$$;
```

### Volatilidade de funções

| Categoria | Significado | Uso |
|---|---|---|
| `IMMUTABLE` | Sempre retorna o mesmo resultado para os mesmos argumentos | Funções matemáticas, formatação |
| `STABLE` | Resultado constante dentro da mesma transação | Consultas ao banco |
| `VOLATILE` (padrão) | Pode retornar resultados diferentes a cada chamada | Funções com efeitos colaterais |

### Procedures (PostgreSQL 11+)

Diferente de funções, procedures podem controlar transações:

```sql
CREATE OR REPLACE PROCEDURE processar_pagamentos()
LANGUAGE plpgsql
AS $$
DECLARE
    r RECORD;
    v_count INT := 0;
BEGIN
    FOR r IN SELECT id, valor_total FROM pedidos WHERE status = 'pendente' LOOP
        UPDATE pedidos SET status = 'processado' WHERE id = r.id;
        INSERT INTO pagamentos (pedido_id, valor) VALUES (r.id, r.valor_total);

        v_count := v_count + 1;

        -- Commit a cada 1000 registros para não segurar lock
        IF v_count % 1000 = 0 THEN
            COMMIT;
        END IF;
    END LOOP;
    COMMIT;
END;
$$;

CALL processar_pagamentos();
```

### Tratamento de exceções

```sql
CREATE OR REPLACE FUNCTION inserir_cliente_seguro(p_nome TEXT, p_email TEXT)
RETURNS BIGINT
LANGUAGE plpgsql
AS $$
DECLARE
    v_id BIGINT;
BEGIN
    INSERT INTO clientes (nome, email) VALUES (p_nome, p_email) RETURNING id INTO v_id;
    RETURN v_id;
EXCEPTION
    WHEN unique_violation THEN
        RAISE NOTICE 'Email % já cadastrado', p_email;
        SELECT id INTO v_id FROM clientes WHERE email = p_email;
        RETURN v_id;
    WHEN others THEN
        RAISE EXCEPTION 'Erro ao inserir cliente: %', SQLERRM;
END;
$$;
```

### Funções com linguagens alternativas

```sql
-- SQL puro (mais performático para queries simples)
CREATE OR REPLACE FUNCTION total_pedidos_cliente(p_id BIGINT)
RETURNS NUMERIC
LANGUAGE sql
STABLE
AS $$
    SELECT COALESCE(SUM(valor_total), 0) FROM pedidos WHERE cliente_id = p_id;
$$;
```

---

## <a id="triggers">15. Triggers</a>

### Trigger function

```sql
-- Trigger para atualizar timestamp automaticamente
CREATE OR REPLACE FUNCTION trigger_atualizar_timestamp()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.atualizado_em = NOW();
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_clientes_atualizado
    BEFORE UPDATE ON clientes
    FOR EACH ROW
    EXECUTE FUNCTION trigger_atualizar_timestamp();
```

### Tipos de trigger

```sql
-- BEFORE INSERT (validação/modificação antes de inserir)
CREATE OR REPLACE FUNCTION validar_email()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.email = lower(trim(NEW.email));
    IF NEW.email !~ '^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$' THEN
        RAISE EXCEPTION 'Email inválido: %', NEW.email;
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_validar_email
    BEFORE INSERT OR UPDATE ON clientes
    FOR EACH ROW
    EXECUTE FUNCTION validar_email();

-- AFTER INSERT (auditoria)
CREATE TABLE auditoria (
    id BIGSERIAL PRIMARY KEY,
    tabela TEXT,
    operacao TEXT,
    dados_antigos JSONB,
    dados_novos JSONB,
    usuario TEXT DEFAULT current_user,
    quando TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION trigger_auditoria()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO auditoria (tabela, operacao, dados_novos)
        VALUES (TG_TABLE_NAME, 'INSERT', to_jsonb(NEW));
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO auditoria (tabela, operacao, dados_antigos, dados_novos)
        VALUES (TG_TABLE_NAME, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW));
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO auditoria (tabela, operacao, dados_antigos)
        VALUES (TG_TABLE_NAME, 'DELETE', to_jsonb(OLD));
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER trg_auditoria_clientes
    AFTER INSERT OR UPDATE OR DELETE ON clientes
    FOR EACH ROW
    EXECUTE FUNCTION trigger_auditoria();

-- INSTEAD OF (para views)
CREATE TRIGGER trg_view_insert
    INSTEAD OF INSERT ON vw_clientes_ativos
    FOR EACH ROW
    EXECUTE FUNCTION fn_view_insert();

-- Trigger condicional (WHEN)
CREATE TRIGGER trg_saldo_negativo
    AFTER UPDATE ON clientes
    FOR EACH ROW
    WHEN (OLD.saldo >= 0 AND NEW.saldo < 0)
    EXECUTE FUNCTION notificar_saldo_negativo();

-- Statement-level trigger (executa uma vez por statement)
CREATE TRIGGER trg_log_truncate
    AFTER TRUNCATE ON clientes
    FOR EACH STATEMENT
    EXECUTE FUNCTION log_truncate();
```

### Gerenciamento de triggers

```sql
-- Desabilitar trigger
ALTER TABLE clientes DISABLE TRIGGER trg_auditoria_clientes;

-- Habilitar trigger
ALTER TABLE clientes ENABLE TRIGGER trg_auditoria_clientes;

-- Desabilitar todos da tabela
ALTER TABLE clientes DISABLE TRIGGER ALL;

-- Remover
DROP TRIGGER IF EXISTS trg_auditoria_clientes ON clientes;
```

---

## <a id="transacoes">16. Transações e Controle de Concorrência (MVCC)</a>

### Transações básicas

```sql
BEGIN;
    UPDATE contas SET saldo = saldo - 100 WHERE id = 1;
    UPDATE contas SET saldo = saldo + 100 WHERE id = 2;
COMMIT;

-- Desfazer
BEGIN;
    DELETE FROM clientes WHERE id = 999;
ROLLBACK;  -- nada foi deletado
```

### Savepoints

```sql
BEGIN;
    INSERT INTO pedidos (...) VALUES (...);

    SAVEPOINT sp1;
    INSERT INTO itens_pedido (...) VALUES (...);  -- pode falhar

    -- Se der erro:
    ROLLBACK TO SAVEPOINT sp1;

    -- Continua com o resto:
    INSERT INTO logs (...) VALUES (...);
COMMIT;
```

### Níveis de isolamento

```sql
-- Definir por transação
BEGIN ISOLATION LEVEL READ COMMITTED;    -- padrão
BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- Definir por sessão
SET default_transaction_isolation = 'serializable';
```

| Nível | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
|---|---|---|---|---|
| Read Committed (padrão) | Não | Possível | Possível | Possível |
| Repeatable Read | Não | Não | Não* | Possível |
| Serializable | Não | Não | Não | Não |

(*) PostgreSQL usa SSI que previne phantoms em Repeatable Read também.

### Locks explícitos

```sql
-- Lock de linha (SELECT FOR UPDATE)
BEGIN;
SELECT * FROM contas WHERE id = 1 FOR UPDATE;
-- Outras transações esperarão para alterar esta linha
UPDATE contas SET saldo = saldo - 100 WHERE id = 1;
COMMIT;

-- FOR UPDATE NOWAIT (erro se já estiver locked)
SELECT * FROM contas WHERE id = 1 FOR UPDATE NOWAIT;

-- FOR UPDATE SKIP LOCKED (ignora linhas locked — útil para filas)
SELECT * FROM tarefas WHERE status = 'pendente'
FOR UPDATE SKIP LOCKED LIMIT 1;

-- FOR SHARE (lock compartilhado — permite leitura mas bloqueia escrita)
SELECT * FROM clientes WHERE id = 1 FOR SHARE;

-- Lock de tabela
LOCK TABLE clientes IN ACCESS EXCLUSIVE MODE;

-- Advisory locks (locks aplicacionais)
SELECT pg_advisory_lock(42);      -- lock global
-- ... operação crítica ...
SELECT pg_advisory_unlock(42);

-- Advisory lock que não espera
SELECT pg_try_advisory_lock(42);  -- retorna true/false
```

### Detectar e resolver deadlocks

```sql
-- Ver locks atuais
SELECT
    pid, usename, state,
    wait_event_type, wait_event,
    query, query_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Ver bloqueios
SELECT
    blocked.pid AS pid_bloqueado,
    blocked.query AS query_bloqueada,
    blocking.pid AS pid_bloqueando,
    blocking.query AS query_bloqueando
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks bk ON bk.locktype = bl.locktype
    AND bk.database IS NOT DISTINCT FROM bl.database
    AND bk.relation IS NOT DISTINCT FROM bl.relation
    AND bk.page IS NOT DISTINCT FROM bl.page
    AND bk.tuple IS NOT DISTINCT FROM bl.tuple
    AND bk.pid != bl.pid
JOIN pg_stat_activity blocking ON blocking.pid = bk.pid
WHERE NOT bl.granted;

-- Cancelar query
SELECT pg_cancel_backend(pid);

-- Terminar conexão
SELECT pg_terminate_backend(pid);
```

### Timeout de locks

```sql
-- Timeout para esperar por lock (evitar espera infinita)
SET lock_timeout = '10s';

-- Timeout de statement
SET statement_timeout = '60s';

-- Timeout de transação ociosa
SET idle_in_transaction_session_timeout = '5min';
```

---

## <a id="jsonb">17. JSON e JSONB</a>

`JSONB` é binário, mais eficiente para consultas; `JSON` armazena texto puro.

### Operações básicas

```sql
CREATE TABLE eventos (
    id BIGSERIAL PRIMARY KEY,
    tipo TEXT,
    dados JSONB DEFAULT '{}'::jsonb,
    criado_em TIMESTAMPTZ DEFAULT NOW()
);

-- Inserir
INSERT INTO eventos (tipo, dados) VALUES (
    'compra',
    '{"produto": "Notebook", "valor": 4500, "cliente": {"nome": "Ana", "email": "ana@email.com"}, "tags": ["eletrônico", "premium"]}'
);
```

### Operadores de acesso

```sql
-- -> retorna JSONB/JSON
SELECT dados->'cliente' FROM eventos;
-- {"nome": "Ana", "email": "ana@email.com"}

-- ->> retorna TEXT
SELECT dados->>'produto' FROM eventos;
-- Notebook

-- Acesso aninhado
SELECT dados->'cliente'->>'nome' FROM eventos;
-- Ana

-- #> caminho como array (retorna JSONB)
SELECT dados #> '{cliente,nome}' FROM eventos;

-- #>> caminho como array (retorna TEXT)
SELECT dados #>> '{cliente,nome}' FROM eventos;
```

### Operadores de consulta

```sql
-- @> contém
SELECT * FROM eventos WHERE dados @> '{"produto": "Notebook"}';

-- <@ está contido em
SELECT * FROM eventos WHERE '{"produto": "Notebook"}'::jsonb <@ dados;

-- ? chave existe
SELECT * FROM eventos WHERE dados ? 'produto';

-- ?| qualquer chave existe
SELECT * FROM eventos WHERE dados ?| array['produto', 'servico'];

-- ?& todas as chaves existem
SELECT * FROM eventos WHERE dados ?& array['produto', 'valor'];

-- Busca em arrays dentro do JSON
SELECT * FROM eventos WHERE dados->'tags' ? 'premium';
```

### Funções de manipulação

```sql
-- Adicionar/alterar chave
UPDATE eventos SET dados = dados || '{"prioridade": "alta"}'::jsonb;

-- Remover chave
UPDATE eventos SET dados = dados - 'prioridade';

-- Remover chave aninhada
UPDATE eventos SET dados = dados #- '{cliente,email}';

-- jsonb_set (alterar valor específico)
UPDATE eventos
SET dados = jsonb_set(dados, '{cliente,nome}', '"Ana Maria"')
WHERE id = 1;

-- jsonb_insert (inserir em array)
UPDATE eventos
SET dados = jsonb_insert(dados, '{tags,0}', '"destaque"')
WHERE id = 1;

-- Construir JSONB
SELECT jsonb_build_object(
    'nome', nome,
    'email', email,
    'saldo', saldo
) FROM clientes;

-- Agregar em JSON
SELECT jsonb_agg(jsonb_build_object('id', id, 'nome', nome))
FROM clientes WHERE ativo;

-- Expandir JSONB em linhas
SELECT * FROM jsonb_each(dados) FROM eventos WHERE id = 1;
SELECT * FROM jsonb_each_text(dados) FROM eventos WHERE id = 1;

-- Expandir array JSONB
SELECT jsonb_array_elements(dados->'tags') FROM eventos;
SELECT jsonb_array_elements_text(dados->'tags') FROM eventos;

-- jsonb_to_record (converter para colunas)
SELECT *
FROM eventos,
     jsonb_to_record(dados) AS x(produto TEXT, valor NUMERIC);

-- jsonb_pretty (formatação legível)
SELECT jsonb_pretty(dados) FROM eventos WHERE id = 1;
```

### jsonpath (PostgreSQL 12+)

```sql
-- Queries com JSONPath
SELECT dados @@ '$.valor > 1000' FROM eventos;

SELECT jsonb_path_query(dados, '$.tags[*]') FROM eventos;

SELECT jsonb_path_query_first(dados, '$.cliente.nome') FROM eventos;

-- Filtro com JSONPath
SELECT jsonb_path_query(dados, '$.tags[*] ? (@ like_regex "^prem")')
FROM eventos;
```

### Índices para JSONB

```sql
-- GIN padrão (suporta @>, ?, ?|, ?&)
CREATE INDEX idx_eventos_dados ON eventos USING gin (dados);

-- GIN jsonb_path_ops (mais compacto, suporta apenas @>)
CREATE INDEX idx_eventos_dados_path ON eventos USING gin (dados jsonb_path_ops);

-- B-tree em expressão específica
CREATE INDEX idx_eventos_produto ON eventos ((dados->>'produto'));
```

---

## <a id="fts">18. Full-Text Search</a>

### Conceitos

- **`tsvector`**: representação do documento (palavras normalizadas com posições).
- **`tsquery`**: representação da busca.
- **Dicionário/Configuração**: define idioma para stemming e stop words.

### Uso básico

```sql
-- Converter texto para tsvector
SELECT to_tsvector('portuguese', 'O PostgreSQL é um banco de dados relacional avançado');
-- 'avanc':8 'banc':5 'dad':7 'postgresql':2 'relacion':8

-- Converter busca para tsquery
SELECT to_tsquery('portuguese', 'banco & dados');
-- 'banc' & 'dad'

-- Operador de match
SELECT * FROM artigos
WHERE to_tsvector('portuguese', titulo || ' ' || conteudo) @@ to_tsquery('portuguese', 'postgresql & avançado');
```

### Coluna tsvector dedicada

```sql
-- Adicionar coluna
ALTER TABLE artigos ADD COLUMN busca_vetor TSVECTOR;

-- Popular
UPDATE artigos
SET busca_vetor = to_tsvector('portuguese', coalesce(titulo, '') || ' ' || coalesce(conteudo, ''));

-- Manter atualizada com trigger
CREATE OR REPLACE FUNCTION atualizar_busca_vetor()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.busca_vetor = to_tsvector('portuguese',
        coalesce(NEW.titulo, '') || ' ' || coalesce(NEW.conteudo, ''));
    RETURN NEW;
END; $$;

CREATE TRIGGER trg_busca_vetor
    BEFORE INSERT OR UPDATE ON artigos
    FOR EACH ROW EXECUTE FUNCTION atualizar_busca_vetor();

-- Índice GIN
CREATE INDEX idx_artigos_busca ON artigos USING gin (busca_vetor);

-- Consultar
SELECT titulo, ts_rank(busca_vetor, q) AS relevancia
FROM artigos, to_tsquery('portuguese', 'banco & dados') q
WHERE busca_vetor @@ q
ORDER BY relevancia DESC;
```

### Pesos e ranking

```sql
-- Pesos: A (maior) a D (menor)
UPDATE artigos
SET busca_vetor =
    setweight(to_tsvector('portuguese', coalesce(titulo, '')), 'A') ||
    setweight(to_tsvector('portuguese', coalesce(resumo, '')), 'B') ||
    setweight(to_tsvector('portuguese', coalesce(conteudo, '')), 'C');

-- Ranking com normalização
SELECT titulo,
       ts_rank_cd(busca_vetor, query, 32) AS rank
FROM artigos, to_tsquery('portuguese', 'postgresql') query
WHERE busca_vetor @@ query
ORDER BY rank DESC;
```

### Busca com trigramas (pg_trgm)

Complementar ao FTS, para buscas por similaridade e typos:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX idx_nome_trgm ON clientes USING gin (nome gin_trgm_ops);

-- Busca por similaridade
SELECT nome, similarity(nome, 'Joao Silba') AS sim
FROM clientes
WHERE nome % 'Joao Silba'
ORDER BY sim DESC;

-- LIKE/ILIKE eficientes com trigramas
SELECT * FROM clientes WHERE nome ILIKE '%silva%';
```

### Operadores de tsquery

```sql
-- AND
to_tsquery('postgresql & banco')

-- OR
to_tsquery('postgresql | mysql')

-- NOT
to_tsquery('banco & !mysql')

-- Frase (adjacência)
phraseto_tsquery('portuguese', 'banco de dados')

-- Busca web-style (converte linguagem natural)
websearch_to_tsquery('portuguese', 'banco de dados -mysql')
```

---

## <a id="particionamento">19. Particionamento de Tabelas</a>

Divide tabelas grandes em partições menores para melhor performance e gerenciamento.

### Particionamento por Range

```sql
CREATE TABLE pedidos (
    id BIGSERIAL,
    cliente_id BIGINT,
    valor_total NUMERIC(12,2),
    criado_em TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, criado_em)
) PARTITION BY RANGE (criado_em);

-- Criar partições
CREATE TABLE pedidos_2025_q1 PARTITION OF pedidos
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

CREATE TABLE pedidos_2025_q2 PARTITION OF pedidos
    FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');

CREATE TABLE pedidos_2025_q3 PARTITION OF pedidos
    FOR VALUES FROM ('2025-07-01') TO ('2025-10-01');

CREATE TABLE pedidos_2025_q4 PARTITION OF pedidos
    FOR VALUES FROM ('2025-10-01') TO ('2026-01-01');

-- Partição default (captura linhas que não encaixam)
CREATE TABLE pedidos_default PARTITION OF pedidos DEFAULT;
```

### Particionamento por Lista

```sql
CREATE TABLE clientes (
    id BIGSERIAL,
    nome TEXT,
    regiao TEXT NOT NULL,
    PRIMARY KEY (id, regiao)
) PARTITION BY LIST (regiao);

CREATE TABLE clientes_sul PARTITION OF clientes
    FOR VALUES IN ('PR', 'SC', 'RS');

CREATE TABLE clientes_sudeste PARTITION OF clientes
    FOR VALUES IN ('SP', 'RJ', 'MG', 'ES');

CREATE TABLE clientes_outras PARTITION OF clientes DEFAULT;
```

### Particionamento por Hash

```sql
CREATE TABLE sessoes (
    id BIGSERIAL,
    token TEXT,
    usuario_id BIGINT NOT NULL,
    PRIMARY KEY (id, usuario_id)
) PARTITION BY HASH (usuario_id);

CREATE TABLE sessoes_p0 PARTITION OF sessoes FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessoes_p1 PARTITION OF sessoes FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessoes_p2 PARTITION OF sessoes FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessoes_p3 PARTITION OF sessoes FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Sub-particionamento

```sql
CREATE TABLE logs (
    id BIGSERIAL,
    nivel TEXT NOT NULL,
    mensagem TEXT,
    criado_em TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, nivel, criado_em)
) PARTITION BY LIST (nivel);

CREATE TABLE logs_error PARTITION OF logs
    FOR VALUES IN ('ERROR', 'CRITICAL')
    PARTITION BY RANGE (criado_em);

CREATE TABLE logs_error_2025_h1 PARTITION OF logs_error
    FOR VALUES FROM ('2025-01-01') TO ('2025-07-01');

CREATE TABLE logs_error_2025_h2 PARTITION OF logs_error
    FOR VALUES FROM ('2025-07-01') TO ('2026-01-01');
```

### Gerenciamento de partições

```sql
-- Desvincular partição (mantém dados, remove da tabela pai)
ALTER TABLE pedidos DETACH PARTITION pedidos_2025_q1;

-- Revincular
ALTER TABLE pedidos ATTACH PARTITION pedidos_2025_q1
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

-- Mover dados antigos para outra tablespace
ALTER TABLE pedidos_2025_q1 SET TABLESPACE arquivo;

-- Dropar partição antiga
DROP TABLE pedidos_2025_q1;

-- Habilitar partition pruning (habilitado por padrão)
SET enable_partition_pruning = on;
```

### Automação de partições

Utilize extensões como `pg_partman` para criar e gerenciar partições automaticamente:

```sql
CREATE EXTENSION pg_partman;

SELECT partman.create_parent(
    p_parent_table := 'public.pedidos',
    p_control := 'criado_em',
    p_type := 'native',
    p_interval := '1 month',
    p_premake := 3
);
```

---

## <a id="performance">20. Performance e Tuning</a>

### EXPLAIN e EXPLAIN ANALYZE

```sql
-- Plano estimado
EXPLAIN SELECT * FROM clientes WHERE email = 'ana@email.com';

-- Plano real com tempos e I/O
EXPLAIN (ANALYZE, BUFFERS, TIMING, FORMAT TEXT)
SELECT * FROM clientes WHERE email = 'ana@email.com';

-- Formato JSON (útil para ferramentas visuais)
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;
```

**Interpretando o plano:**

| Nó | Significado |
|---|---|
| `Seq Scan` | Leitura sequencial (full scan) |
| `Index Scan` | Busca por índice + consulta heap |
| `Index Only Scan` | Dados vêm apenas do índice (ideal) |
| `Bitmap Index/Heap Scan` | Combinação de índice bitmap + heap |
| `Nested Loop` | Join por loop aninhado |
| `Hash Join` | Join com hash em memória |
| `Merge Join` | Join com merge de listas ordenadas |
| `Sort` | Ordenação (pode usar disco se `work_mem` insuficiente) |
| `Aggregate` | Agregação (COUNT, SUM, etc.) |

### pg_stat_statements

```sql
-- Habilitar (requer restart)
-- Em postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.track = all

CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 queries mais demoradas
SELECT
    substring(query, 1, 80) AS query_resumo,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS media_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries com mais I/O
SELECT
    substring(query, 1, 80),
    shared_blks_read + shared_blks_hit AS total_buffers,
    round(100.0 * shared_blks_hit / NULLIF(shared_blks_read + shared_blks_hit, 0), 2) AS hit_ratio
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 10;

-- Resetar estatísticas
SELECT pg_stat_statements_reset();
```

### Cache hit ratio

```sql
-- Ratio geral (ideal > 99%)
SELECT
    sum(blks_hit) AS cache_hit,
    sum(blks_read) AS disk_read,
    round(sum(blks_hit)::numeric / (sum(blks_hit) + sum(blks_read)) * 100, 2) AS hit_ratio
FROM pg_stat_database;

-- Por tabela
SELECT
    relname,
    heap_blks_hit,
    heap_blks_read,
    CASE WHEN heap_blks_hit + heap_blks_read > 0
        THEN round(100.0 * heap_blks_hit / (heap_blks_hit + heap_blks_read), 2)
        ELSE 0 END AS hit_ratio
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 20;
```

### Tamanhos de tabelas e índices

```sql
-- Tamanho de tabelas (com e sem índices)
SELECT
    relname AS tabela,
    pg_size_pretty(pg_total_relation_size(relid)) AS total,
    pg_size_pretty(pg_relation_size(relid)) AS dados,
    pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) AS indices_toast
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Tamanho de índices
SELECT
    indexrelname AS indice,
    relname AS tabela,
    pg_size_pretty(pg_relation_size(indexrelid)) AS tamanho
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Bloat estimado de tabelas
SELECT
    schemaname, relname,
    n_live_tup, n_dead_tup,
    CASE WHEN n_live_tup > 0
        THEN round(100.0 * n_dead_tup / n_live_tup, 2)
        ELSE 0 END AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### Dicas de tuning (resumo)

| Parâmetro | Regra geral |
|---|---|
| `shared_buffers` | 25% da RAM |
| `effective_cache_size` | 50-75% da RAM |
| `work_mem` | Cuidado: multiplicado por conexões × operações |
| `maintenance_work_mem` | 512MB a 2GB |
| `random_page_cost` | 1.1 para SSD, 4.0 para HDD |
| `effective_io_concurrency` | 200 para SSD, 2 para HDD |
| `max_parallel_workers_per_gather` | 2-4 (para parallelism) |
| `huge_pages` | `try` se o SO suportar |

> Use **pgtune** (https://pgtune.leopard.in.ua/) para gerar configurações iniciais baseadas no hardware.

---

## <a id="backup">21. Backup e Restore</a>

### Backup lógico (pg_dump)

```bash
# Dump em formato custom (compactado, mais flexível)
pg_dump -U postgres -Fc -f meu_bd.dump minha_base

# Dump em SQL puro
pg_dump -U postgres -Fp -f meu_bd.sql minha_base

# Dump apenas schema (sem dados)
pg_dump -U postgres --schema-only -f schema.sql minha_base

# Dump apenas dados
pg_dump -U postgres --data-only -f dados.sql minha_base

# Dump de tabelas específicas
pg_dump -U postgres -t clientes -t pedidos -Fc -f parcial.dump minha_base

# Dump de um schema específico
pg_dump -U postgres -n vendas -Fc -f vendas.dump minha_base

# Dump de todas as bases
pg_dumpall -U postgres -f tudo.sql
```

### Restore

```bash
# Restore de formato custom
pg_restore -U postgres -d minha_base -j 4 meu_bd.dump

# Restore criando o banco
pg_restore -U postgres -C -d postgres meu_bd.dump

# Restore apenas de uma tabela
pg_restore -U postgres -t clientes -d minha_base meu_bd.dump

# Restore de SQL puro
psql -U postgres -d minha_base -f meu_bd.sql

# Listar conteúdo do dump
pg_restore -l meu_bd.dump
```

### Backup físico (pg_basebackup)

```bash
# Base backup completo
pg_basebackup -h localhost -U replicator -D /backup/base -Fp -Xs -P

# Compactado
pg_basebackup -h localhost -U replicator -D /backup/base -Ft -z -Xs -P
```

### PITR (Point-in-Time Recovery)

1. Configure archiving em `postgresql.conf`:

```conf
archive_mode = on
archive_command = 'cp %p /mnt/wal_archive/%f'
```

2. Após um desastre, restaure e configure `postgresql.conf`:

```conf
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2026-02-28 14:30:00'
```

3. Crie o arquivo sinalizador e inicie:

```bash
touch $PGDATA/recovery.signal
sudo systemctl start postgresql
```

### Ferramentas de backup em produção

- **pgBackRest**: backup incremental, compressão, retenção, verificação.
- **Barman**: backup e recovery gerenciados pela 2ndQuadrant/EDB.
- **WAL-G**: archiving para object storage (S3, GCS, Azure).

---

## <a id="replicacao">22. Replicação</a>

### Streaming Replication (física)

**No primary** (`postgresql.conf`):

```conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = '1GB'          # ou wal_keep_segments (antigo)
archive_mode = on
archive_command = 'cp %p /mnt/wal_archive/%f'
```

**Criar role de replicação:**

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'senha_segura';
```

**Permitir em `pg_hba.conf`:**

```conf
host replication replicator 10.0.0.0/24 scram-sha-256
```

**No standby:**

```bash
# Base backup
pg_basebackup -h primary -D /var/lib/postgresql/16/main -U replicator -Fp -Xs -P

# Criar arquivo sinalizador
touch /var/lib/postgresql/16/main/standby.signal
```

Adicionar ao `postgresql.conf` do standby:

```conf
primary_conninfo = 'host=primary port=5432 user=replicator password=senha_segura'
```

Iniciar o standby:

```bash
sudo systemctl start postgresql
```

### Replicação síncrona

No primary:

```conf
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
# ou 'ANY 1 (standby1, standby2)' para quorum
```

### Replicação lógica

```sql
-- No publisher
ALTER SYSTEM SET wal_level = 'logical';
-- Restart necessário

CREATE PUBLICATION pub_vendas FOR TABLE clientes, pedidos;
-- Ou todas: CREATE PUBLICATION pub_all FOR ALL TABLES;

-- No subscriber
CREATE SUBSCRIPTION sub_vendas
    CONNECTION 'host=publisher dbname=app_db user=replicator password=senha'
    PUBLICATION pub_vendas;
```

### Monitoramento de replicação

```sql
-- No primary
SELECT
    client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- No standby
SELECT
    now() - pg_last_xact_replay_timestamp() AS lag_tempo;
```

### Failover

```bash
# Promover standby a primary
pg_ctl promote -D /var/lib/postgresql/16/main
# Ou: SELECT pg_promote();
```

Para failover automatizado, use **Patroni** ou **repmgr**.

---

## <a id="seguranca">23. Segurança</a>

### SSL/TLS

Em `postgresql.conf`:

```conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'              # para verificação de clientes
ssl_min_protocol_version = 'TLSv1.2'
```

Em `pg_hba.conf`, forçar SSL:

```conf
hostssl all all 0.0.0.0/0 scram-sha-256
```

Conexão com SSL:

```bash
psql "postgresql://user:pass@host:5432/db?sslmode=verify-full&sslrootcert=/path/ca.crt"
```

### Row Level Security (RLS)

```sql
-- Habilitar RLS na tabela
ALTER TABLE documentos ENABLE ROW LEVEL SECURITY;

-- Política: cada usuário vê apenas seus documentos
CREATE POLICY docs_owner_policy ON documentos
    USING (proprietario = current_user);

-- Política separada para INSERT
CREATE POLICY docs_insert_policy ON documentos
    FOR INSERT
    WITH CHECK (proprietario = current_user);

-- Política para role específica
CREATE POLICY docs_admin_policy ON documentos
    FOR ALL
    TO role_admin
    USING (true)
    WITH CHECK (true);

-- Forçar RLS mesmo para o owner da tabela
ALTER TABLE documentos FORCE ROW LEVEL SECURITY;
```

### Criptografia de dados

```sql
-- pgcrypto para criptografia em colunas
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Hash de senha
INSERT INTO usuarios (email, senha_hash)
VALUES ('ana@email.com', crypt('minha_senha', gen_salt('bf', 12)));

-- Verificar senha
SELECT *
FROM usuarios
WHERE email = 'ana@email.com'
  AND senha_hash = crypt('minha_senha', senha_hash);

-- Criptografia simétrica
INSERT INTO dados_sensiveis (cpf_enc)
VALUES (pgp_sym_encrypt('12345678901', 'chave_secreta'));

SELECT pgp_sym_decrypt(cpf_enc, 'chave_secreta') FROM dados_sensiveis;
```

### Auditoria

- Use a extensão **pgAudit** para log detalhado de operações:

```conf
# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'ddl, write'
```

```sql
CREATE EXTENSION pgaudit;
```

### Boas práticas de segurança

- **Nunca** use `trust` em `pg_hba.conf` em produção.
- Use `scram-sha-256` em vez de `md5`.
- Defina `password_encryption = 'scram-sha-256'` em `postgresql.conf`.
- Limite `max_connections` e use **connection pooling** (PgBouncer).
- Defina `statement_timeout` e `idle_in_transaction_session_timeout`.
- Mantenha PostgreSQL atualizado (patches de segurança).
- Revise permissões regularmente (princípio do menor privilégio).
- Desabilite a extensão `dblink` se não for necessária.

---

## <a id="monitoramento">24. Monitoramento e Manutenção</a>

### VACUUM

```sql
-- Vacuum manual (recupera espaço, atualiza visibility map)
VACUUM clientes;

-- Vacuum verbose
VACUUM (VERBOSE) clientes;

-- Vacuum Full (reescreve a tabela — lock exclusivo!)
VACUUM FULL clientes;

-- Vacuum + Analyze
VACUUM ANALYZE clientes;
```

### ANALYZE

```sql
-- Atualizar estatísticas de uma tabela
ANALYZE clientes;

-- Atualizar todas
ANALYZE;

-- Verbose
ANALYZE (VERBOSE) clientes;
```

### Configuração do Autovacuum

```sql
-- Por tabela (tabelas com muito churn)
ALTER TABLE logs SET (
    autovacuum_vacuum_scale_factor = 0.01,     -- vacuum quando 1% morreu
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_vacuum_cost_delay = '0ms'        -- sem throttle
);
```

### Views de monitoramento essenciais

```sql
-- Sessões ativas
SELECT pid, usename, datname, state, query, query_start,
       now() - query_start AS duracao
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duracao DESC;

-- Queries idle in transaction (perigosas)
SELECT pid, usename, state, query, now() - xact_start AS tx_duracao
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY tx_duracao DESC;

-- Conexões por estado
SELECT state, COUNT(*) FROM pg_stat_activity GROUP BY state;

-- Tabelas com mais operações
SELECT relname, seq_scan, idx_scan, n_tup_ins, n_tup_upd, n_tup_del, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_tup_ins + n_tup_upd + n_tup_del DESC
LIMIT 20;

-- Tabelas que precisam de vacuum
SELECT relname, last_vacuum, last_autovacuum, n_dead_tup,
       n_live_tup, round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;

-- Uso de disco por tabela
SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(relid)) AS total,
    pg_size_pretty(pg_relation_size(relid)) AS tabela,
    pg_size_pretty(pg_indexes_size(relid)) AS indices
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Checkpoint stats
SELECT * FROM pg_stat_bgwriter;
```

### REINDEX e pg_repack

```sql
-- Reindex (lock exclusivo no índice)
REINDEX INDEX idx_clientes_email;
REINDEX TABLE clientes;

-- Reindex concorrente (PostgreSQL 12+)
REINDEX (VERBOSE) TABLE CONCURRENTLY clientes;
```

Para reduzir bloat sem lock exclusivo, use **pg_repack**:

```bash
pg_repack -d meu_bd -t clientes
```

### Connection pooling (PgBouncer)

Arquivo `pgbouncer.ini`:

```ini
[databases]
app_db = host=127.0.0.1 port=5432 dbname=app_db

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
```

### pg_stat_activity — diagnosticar problemas

```sql
-- Matar query lenta
SELECT pg_cancel_backend(pid);    -- cancela query
SELECT pg_terminate_backend(pid); -- mata conexão

-- Matar todas as conexões de um banco (para DROP DATABASE)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'meu_bd' AND pid <> pg_backend_pid();
```

---

## <a id="extensoes">25. Extensões Úteis</a>

```sql
-- Listar extensões disponíveis
SELECT * FROM pg_available_extensions ORDER BY name;

-- Listar instaladas
\dx
```

| Extensão | Descrição |
|---|---|
| `pg_stat_statements` | Estatísticas de queries (essencial) |
| `pgcrypto` | Funções de criptografia |
| `uuid-ossp` | Geração de UUID |
| `hstore` | Pares chave-valor |
| `pg_trgm` | Busca por trigramas (similaridade) |
| `btree_gist` / `btree_gin` | Operadores extras para GiST/GIN |
| `citext` | Tipo TEXT case-insensitive |
| `tablefunc` | Funcões de pivô (crosstab) |
| `pgaudit` | Auditoria detalhada |
| `pg_partman` | Gerenciamento automático de partições |
| `pg_repack` | Reorgarnização de tabelas sem lock |
| `pg_cron` | Agendamento de jobs dentro do PostgreSQL |
| `postgis` | Dados geoespaciais |
| `timescaledb` | Séries temporais |
| `pg_hint_plan` | Forçar planos de execução |
| `auto_explain` | Log automático de planos lentos |
| `postgres_fdw` | Acesso a outros servidores PostgreSQL |
| `file_fdw` | Acesso a arquivos CSV como tabelas |
| `dblink` | Queries entre bancos |
| `pg_stat_kcache` | Estatísticas de CPU e I/O por query |

### Exemplos de uso

```sql
-- pg_cron: agendar vacuum diário
CREATE EXTENSION pg_cron;
SELECT cron.schedule('vacuum-diario', '0 3 * * *', 'VACUUM ANALYZE clientes');

-- auto_explain em postgresql.conf
-- shared_preload_libraries = 'auto_explain'
-- auto_explain.log_min_duration = '1s'
-- auto_explain.log_analyze = true

-- postgres_fdw: acessar outro servidor
CREATE EXTENSION postgres_fdw;
CREATE SERVER srv_remoto FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host '10.0.0.5', dbname 'outro_bd', port '5432');
CREATE USER MAPPING FOR app_user SERVER srv_remoto
    OPTIONS (user 'remote_user', password 'senha');
IMPORT FOREIGN SCHEMA public FROM SERVER srv_remoto INTO schema_remoto;

-- citext: texto case-insensitive
CREATE EXTENSION citext;
CREATE TABLE usuarios (
    email CITEXT UNIQUE
);
INSERT INTO usuarios VALUES ('Ana@Email.com');
SELECT * FROM usuarios WHERE email = 'ana@email.com';  -- encontra
```

---

## <a id="upgrade">26. Upgrade de Versão</a>

### Minor upgrade (ex: 16.1 → 16.4)

```bash
# Atualizar pacotes (mantém dados intactos)
sudo apt update && sudo apt upgrade postgresql-16
sudo systemctl restart postgresql
```

### Major upgrade (ex: 15 → 16)

**Opção 1: pg_upgrade (mais rápido)**

```bash
# 1. Instalar nova versão
sudo apt install postgresql-16

# 2. Parar ambas
sudo systemctl stop postgresql

# 3. Executar pg_upgrade
sudo -u postgres /usr/lib/postgresql/16/bin/pg_upgrade \
    --old-datadir=/var/lib/postgresql/15/main \
    --new-datadir=/var/lib/postgresql/16/main \
    --old-bindir=/usr/lib/postgresql/15/bin \
    --new-bindir=/usr/lib/postgresql/16/bin \
    --check    # primeiro teste com --check

# 4. Se o check passar, executar sem --check
sudo -u postgres /usr/lib/postgresql/16/bin/pg_upgrade \
    --old-datadir=/var/lib/postgresql/15/main \
    --new-datadir=/var/lib/postgresql/16/main \
    --old-bindir=/usr/lib/postgresql/15/bin \
    --new-bindir=/usr/lib/postgresql/16/bin \
    --link     # --link usa hard links (muito mais rápido)

# 5. Iniciar nova versão
sudo systemctl start postgresql@16-main

# 6. Atualizar estatísticas
/usr/lib/postgresql/16/bin/vacuumdb --all --analyze-in-stages

# 7. Remover versão antiga (quando confortável)
sudo apt remove postgresql-15
```

**Opção 2: pg_dump/pg_restore (mais seguro, mais lento)**

```bash
pg_dumpall -U postgres -h old_host -p 5432 | psql -U postgres -h new_host -p 5432
```

**Opção 3: Replicação lógica (zero-downtime)**

1. Configurar replicação lógica do PG 15 para PG 16.
2. Aguardar sincronização.
3. Redirecionar aplicações para o novo servidor.
4. Desativar replicação.

---

## <a id="referencia">27. Referência Rápida de Comandos</a>

### Administrativos (shell)

```bash
# Iniciar/parar/reiniciar
sudo systemctl start postgresql
sudo systemctl stop postgresql
sudo systemctl restart postgresql
sudo systemctl reload postgresql    # recarrega config sem parar

# Verificar status
sudo systemctl status postgresql
pg_isready -h localhost -p 5432

# Localizar arquivos de configuração
sudo -u postgres psql -c "SHOW config_file;"
sudo -u postgres psql -c "SHOW hba_file;"
sudo -u postgres psql -c "SHOW data_directory;"

# Logs
sudo journalctl -u postgresql -f
tail -f /var/log/postgresql/postgresql-16-main.log
```

### Informações do servidor (SQL)

```sql
SELECT version();                           -- Versão completa
SELECT current_database();                  -- Banco atual
SELECT current_user;                        -- Usuário atual
SELECT current_schema();                    -- Schema atual
SELECT inet_server_addr(), inet_server_port();  -- IP e porta
SHOW server_version;
SHOW all;                                   -- Todos os parâmetros

-- Uptime
SELECT pg_postmaster_start_time();
SELECT now() - pg_postmaster_start_time() AS uptime;
```

### Utilitários de linha de comando

| Comando | Descrição |
|---|---|
| `psql` | Cliente interativo |
| `pg_dump` | Backup lógico de um banco |
| `pg_dumpall` | Backup de todos os bancos + roles |
| `pg_restore` | Restore de dumps no formato custom/tar |
| `pg_basebackup` | Backup físico |
| `pg_isready` | Verificar se o servidor aceita conexões |
| `pg_ctl` | Controle do servidor |
| `pg_upgrade` | Upgrade de major version |
| `createdb` / `dropdb` | Criar/remover banco |
| `createuser` / `dropuser` | Criar/remover role |
| `vacuumdb` | Executar VACUUM |
| `reindexdb` | Reindexar banco |
| `clusterdb` | Clusterizar tabelas |
| `pg_config` | Informações de compilação |

---

**Referências:**

- [Documentação Oficial do PostgreSQL](https://www.postgresql.org/docs/current/)
- [PostgreSQL Wiki](https://wiki.postgresql.org/)
- [PGTune](https://pgtune.leopard.in.ua/)
- [pgBadger](https://pgbadger.darold.net/)
- [explain.depesz.com](https://explain.depesz.com/)
