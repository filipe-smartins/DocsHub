# Tuning de Índices — tipos, estruturas e casos de uso

Índices são cruciais para desempenho de consultas; escolher o tipo e a configuração correta evita leituras completas desnecessárias.

## Tipos de índices (PostgreSQL)

- **B-tree**: padrão para comparações (=, <, >, BETWEEN, ORDER BY).
- **GIN**: para consultas em arrays e JSONB, full-text search (`gin_trgm_ops`).
- **GiST**: indexação espacial e range indexes.
- **BRIN**: eficiente para colunas com localização física correlacionada (timestamp em tabelas append-only).

## Índices compostos e ordem

- Ordem das colunas importa: `CREATE INDEX ON t (a, b)` é útil para filtros por `a` e por `a,b`.
- Prefira colocar colunas com maior seletividade primeiro.

## Índices parciais e expressões

- Índices parciais (`WHERE ativo = true`) reduzem tamanho e mantêm desempenho quando aplicável.
- Índices de expressão (`CREATE INDEX ON t (lower(email))`) tornam consultas case-insensitive eficientes.

## Cobertura (covering indexes)

- Projetar índices que incluam todas as colunas necessárias (`INCLUDE` no Postgres) permite index-only scans.

```sql
CREATE INDEX idx_orders_customer_total ON orders (customer_id) INCLUDE (total);
```

## Fillfactor e manutenção

- `FILLFACTOR` controla quanto espaço livre manter para updates; útil em índices com muitas atualizações.
- Rotinas: `VACUUM`, `ANALYZE`, `REINDEX` e `pg_repack` para reduzir bloat.

## Cuidado com excesso de índices

- Cada índice aumenta custo de inserts/updates/deletes; crie índices com base em queries reais (pg_stat_statements).

## Diagnóstico

- Use `EXPLAIN (ANALYZE, BUFFERS)` para ver uso de índices e custos.
- Verifique `pg_stat_user_indexes` e `pg_stat_user_tables` para uso e eficiência.

## Exemplos práticos

- Index para queries por data em grandes tabelas: usar `BRIN` se inserções forem append-only.
- Index GIN para JSONB:

```sql
CREATE INDEX idx_jsonb_data ON events USING gin (data jsonb_path_ops);
```

Referências: documentação do PostgreSQL sobre índices e tuning guides.
