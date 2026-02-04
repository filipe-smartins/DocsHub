# SQL - INSERT (Inserir registros)

`INSERT` adiciona novos registros a uma tabela.

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Sintaxe básica:

```sql
INSERT INTO tabela (col1, col2) VALUES (val1, val2);
```

Inserir múltiplas linhas:

```sql
INSERT INTO usuarios (nome, email) VALUES
	('Ana', 'ana@x.com'),
	('Bruno', 'bruno@x.com');
```

Inserir com `SELECT`:

```sql
INSERT INTO clientes_ativos (id, nome)
SELECT id, nome FROM clientes WHERE ativo = true;
```

PostgreSQL: `RETURNING` para obter a linha criada:

```sql
INSERT INTO pedidos (cliente_id, total) VALUES (1, 100) RETURNING id, created_at;
```

Upsert (Postgres):

```sql
INSERT INTO tabela (id, valor) VALUES (1, 'x')
ON CONFLICT (id) DO UPDATE SET valor = EXCLUDED.valor;
```

Boas práticas:
- Valide dados antes de inserir (constraints, checagens).
- Use transações para operações múltiplas.

