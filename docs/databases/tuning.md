# Bancos de Dados Avançado — Tuning de índices e queries

Este guia resume técnicas para indexação eficiente e otimização de consultas (ex.: PostgreSQL).

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Identificando problemas

- Use `EXPLAIN` / `EXPLAIN ANALYZE` para ver o plano e tempos reais.
- `pg_stat_statements` ajuda a encontrar queries mais custosas.

Tipos de índices e quando usar

- `btree` (padrão): operadores de comparação (=, <, >, BETWEEN, ORDER BY).
- `hash`: alguns casos de igualdade (pouco usado atualmente).
- `gin` / `gist`: colunas JSONB, arrays, full-text search.
- Indexes parciais: indexar apenas linhas que interessam (`WHERE ativo = true`).
- Indexes de expressão: indexar `lower(email)` para buscas case-insensitive.

Boas práticas de criação

- Indexe colunas usadas em `WHERE`, `JOIN` e `ORDER BY` frequentemente.
- Evite indexar colunas com pouca seletividade (booleans com 90% true).
- Combine colunas em índices compostos na ordem usada nas queries.

Manutenção e estatísticas

- `ANALYZE` atualiza estatísticas; `VACUUM` recupera espaço e mantém desempenho.
- `autovacuum` deve estar ativo; ajuste `autovacuum_vacuum_scale_factor` e `threshold` se necessário.
- Reindex quando índices ficarem muito fragmentados: `REINDEX TABLE tabela;`.

Otimização de queries

- Reescreva queries para permitir uso de índices (evite funções na coluna sem index correspondente).
- Substitua `SELECT *` por colunas necessárias.
- Considere materialized views para consultas agregadas pesadas.

Paralelismo e recursos

- Ajuste `work_mem` para operações de sort/hash que ocorrem em queries pesadas.
- `shared_buffers`, `effective_cache_size` e `max_worker_processes` influenciam planejamento.

Diagnóstico rápido

```sql
EXPLAIN ANALYZE SELECT ...;
-- Ver queries mais lentas
SELECT query, total_time, calls FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;
```

Evitar armadilhas

- Não crie índices em excesso — cada insert/update paga o custo do índice.
- Não confie apenas em índices; resolva problemas de I/O, locks e contenda também.

Ferramentas úteis

- `pg_stat_activity`, `pg_locks`, `pg_stat_user_indexes`, `pg_stat_user_tables`.
- Ferramentas externas: pgBadger (análise de logs), pgtune (sugestões de parâmetros), explain.depesz.com para visualização de planos.
