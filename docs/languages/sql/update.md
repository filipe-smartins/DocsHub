# SQL - UPDATE (Atualizar registros)

O comando `UPDATE` modifica linhas existentes em uma tabela. Sempre use `WHERE` para evitar atualizar todas as linhas por engano.

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Sintaxe básica:

```sql
UPDATE nome_tabela
SET coluna1 = valor1, coluna2 = valor2
WHERE condicao;
```

Exemplo:

```sql
UPDATE clientes
SET ativo = false
WHERE ultimo_login < '2020-01-01';
```

Usando `RETURNING` no PostgreSQL para ver os registros alterados:

```sql
UPDATE contas
SET saldo = saldo - 100
WHERE id = 1
RETURNING id, saldo;
```

Transações (sempre que modificar múltiplas tabelas):

```sql
BEGIN;
UPDATE contas SET saldo = saldo - 100 WHERE id = 1;
UPDATE contas SET saldo = saldo + 100 WHERE id = 2;
COMMIT;
```

Boas práticas:
- Verificar com `SELECT` antes de `UPDATE` para confirmar o conjunto de linhas.
- Fazer backup ou usar transações em operações críticas.
- Atualizar em lotes (`LIMIT`/`OFFSET` ou condições por faixa) quando precisar processar muitas linhas.
