# Backups Avançados — estratégias, PITR e ferramentas

Backups robustos são essenciais para RPO/RTO. Aqui focamos em bancos relacionais (PostgreSQL) e estratégias avançadas.

## Tipos de backup

- **Backup lógico**: `pg_dump` / `pg_dumpall` (útil para migrações e restores por objeto).
- **Backup físico**: base backups (`pg_basebackup`) + WAL archiving para PITR.

## Point-in-Time Recovery (PITR)

- Configure `archive_mode = on` e um `archive_command` que salve arquivos WAL externamente.
- Fluxo: base backup consistente + arquivos WAL arquivados → restaurar base e aplicar WAL até o momento desejado.

## Ferramentas de produção

- `pgbackrest` e `barman` — soluções completas para backups físicos, retenção, compressão e restauração.
- `wal-e` / `wal-g` — soluções para arquivar WALs em object storage (S3).

## Estratégias e políticas

- Combine backups físicos (rápido restore) com dumps lógicos (migração, verificação).
- Tenha retenção e rotação: políticas de retenção por dia/semana/mês.
- Automatize verificações de integridade (restore em ambiente isolado) regularmente.

## Backup em ambiente containerizado

- Não confie em snapshots de container para bancos de dados sem garantir quiesce/consistência; preferir base backups e dumps.

## Testing e documentação

- Documente procedimentos de restore e teste com frequência (drills).
- Meça RTO (tempo para ficar online) e RPO (ponto máximo de perda aceitável) e ajuste janelas e frequência.

## Exemplo rápido — base backup + WAL

```bash
pg_basebackup -h primary -D /backup/base -U replicator -Fp -Xs -P
# archive_command em postgresql.conf -> cp %p /mnt/wal_archive/%f
```

Referências: documentação PostgreSQL, guias de `pgbackrest` e `barman`.
