# Bancos de Dados Avançado — Replicação

Visão geral de replicação de bancos (ex.: PostgreSQL streaming replication).

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Tipos comuns
- Replicação física (streaming WAL) — replica bytes do WAL, cópia exata do cluster.
- Replicação lógica — replica mudanças em nível de tabela; permite replicação seletiva e transformações.

Configuração básica (streaming physical replication)

1. No `postgresql.conf` (primary):

```conf
wal_level = replica
max_wal_senders = 5
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
```

2. Criar usuário de replicação no primary:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'senha_segura';
```

3. Permitir no `pg_hba.conf`:

```text
host replication replicator replica_ip/32 md5
```

4. Base backup para o standby:

```bash
pg_basebackup -h primary -D /var/lib/postgresql/12/main -U replicator -Fp -Xs -P
```

5. Configurar `standby.signal` e `primary_conninfo` (Postgres >=12) no standby.

Failover e promoção

- Para promover um standby a primary: `pg_ctl promote` ou criar `promote.signal`.
- Use ferramentas de orquestração (repmgr, Patroni) para gerenciamento de failover automatizado e quarentena de split-brain.

Síncrono vs Assíncrono

- Assíncrono: menor latência no primary; risco de perda de commits recentes em failover.
- Síncrono: garante que commit reflita em replica(s) escolhidas; aumenta latência.

Monitoramento

- Verificação de lag: `pg_stat_replication` no primary.
- Monitorar WAL archive, tamanho de retentativas e conexões.

Boas práticas

- Configure retenção de WAL suficiente para recuperação.
- Teste failover em ambiente controlado.
- Automatize e monitore com ferramentas (Prometheus exporters, repmgr, Patroni).
