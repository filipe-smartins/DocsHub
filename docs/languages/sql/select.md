# SQL - SELECT (Consultar dados)

O `SELECT` é usado para ler dados de uma ou mais tabelas.

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Exemplo básico:

```sql
SELECT coluna1, coluna2 FROM tabela WHERE condicao ORDER BY coluna1 DESC LIMIT 10;
```

Cláusulas comuns:
- `WHERE` — filtra linhas.
- `ORDER BY` — ordena resultados.
- `LIMIT` — limita número de linhas retornadas.
- `JOIN` — combina tabelas:

```sql
SELECT c.nome, p.total
FROM clientes c
INNER JOIN pedidos p ON p.cliente_id = c.id
```

Agregações e `GROUP BY`:

```sql
SELECT status, COUNT(*) AS total FROM pedidos GROUP BY status HAVING COUNT(*) > 5;
```

Dicas:
- Evite `SELECT *` em produção; especifique colunas.
- Use índices para melhorar `WHERE` e `JOIN`.
- Analise planos com `EXPLAIN`/`EXPLAIN ANALYZE`.

