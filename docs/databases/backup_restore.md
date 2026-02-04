# Bancos de Dados Avançado — Backup e Restore

Este guia foca em estratégias de backup/restore para bancos relacionais (ex.: PostgreSQL).

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Backups lógicos (pg_dump / pg_dumpall)

```bash
# Dump de uma base específica (SQL):
pg_dump -U postgres -F p -f meu_bd.sql minha_base

# Dump compactado em formato custom (ideal para pg_restore):
pg_dump -U postgres -F c -f meu_bd.dump minha_base

# Dump de todas as bases:
pg_dumpall -U postgres -f todas_bases.sql
```

Restore de backups lógicos:

```bash
# SQL plain:
psql -U postgres -d minha_base -f meu_bd.sql

# formato custom com pg_restore (permite paralelismo):
createdb -U postgres nova_base
pg_restore -U postgres -d nova_base -j 4 meu_bd.dump
```

Backups físicos (base backups e WAL archiving) — para RPO/RTO menores

- Use `pg_basebackup` para criar uma cópia física consistente:

```bash
pg_basebackup -h primary -D /var/lib/postgresql/backup -U replicator -Fp -Xs -P
```

- Configure `archive_mode = on` e `archive_command` para arquivar WALs, permitindo PITR (Point-in-Time Recovery).

Exemplo de recovery (PITR):

1. Pare o servidor.
2. Restaure o base backup em `PGDATA`.
3. Coloque arquivos WAL arquivados em `pg_wal` ou configure `restore_command` para buscá-los.
4. Crie `recovery.signal` (Postgres >=12) e ajuste `recovery_target_time` se necessário.

Testes e automação

- Teste seus backups regularmente: `pg_restore --test` ou restaurando em outro ambiente.
- Automatize com scripts e monitore tamanho/idade dos backups.
- Faça retenção (rotate) e verifique integridade (checksums/restore test).

Boas práticas

- Combine backups lógicos (para migrações/exports) e físicos (para recuperação rápida).  
- Documente procedimentos de restauração e verifique os tempos de recuperação.  
- Proteja cópias de backup (criptografia em trânsito e em repouso).
