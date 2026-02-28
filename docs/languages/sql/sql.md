# SQL: Guia Completo — do Básico ao Avançado

Guia abrangente da linguagem SQL cobrindo consultas, manipulação de dados, definição de estruturas, controle de acesso, otimização e técnicas avançadas. Os exemplos usam sintaxe padrão (SQL:2016) com notas para PostgreSQL onde aplicável.

---

## Índice

- [1. Fundamentos](#fundamentos)
- [2. SELECT — Consultas](#select)
- [3. WHERE — Filtragem](#where)
- [4. ORDER BY, LIMIT e OFFSET](#order-limit)
- [5. JOIN — Combinação de tabelas](#joins)
- [6. Agregações e GROUP BY](#agregacoes)
- [7. HAVING](#having)
- [8. Subqueries](#subqueries)
- [9. CTEs (Common Table Expressions)](#ctes)
- [10. Window Functions](#window-functions)
- [11. UNION, INTERSECT e EXCEPT](#set-operations)
- [12. INSERT](#insert)
- [13. UPDATE](#update)
- [14. DELETE e TRUNCATE](#delete)
- [15. UPSERT (INSERT ON CONFLICT / MERGE)](#upsert)
- [16. DDL — CREATE, ALTER, DROP](#ddl)
- [17. Tipos de Dados](#tipos-dados)
- [18. Constraints](#constraints)
- [19. Índices](#indices)
- [20. Views e Materialized Views](#views)
- [21. Transações](#transacoes)
- [22. DCL — Permissões (GRANT/REVOKE)](#dcl)
- [23. Funções e Expressões Úteis](#funcoes)
- [24. CASE, COALESCE e NULLIF](#case-coalesce)
- [25. Expressões Regulares e Pattern Matching](#regex)
- [26. JSON/JSONB](#json)
- [27. EXPLAIN — Análise de Planos](#explain)
- [28. Boas Práticas e Otimização](#boas-praticas)

---

## <a id="fundamentos">1. Fundamentos</a>

### Categorias de comandos SQL

| Categoria | Comandos | Descrição |
|---|---|---|
| **DQL** (Query) | `SELECT` | Consultar dados |
| **DML** (Manipulation) | `INSERT`, `UPDATE`, `DELETE`, `MERGE` | Manipular dados |
| **DDL** (Definition) | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` | Definir estruturas |
| **DCL** (Control) | `GRANT`, `REVOKE` | Controle de acesso |
| **TCL** (Transaction) | `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` | Controle transacional |

### Ordem lógica de execução do SELECT

Embora escrevamos `SELECT ... FROM ... WHERE ...`, a ordem **lógica** de processamento é:

```
1. FROM / JOIN          → Define as tabelas e junções
2. WHERE                → Filtra linhas
3. GROUP BY             → Agrupa linhas
4. HAVING               → Filtra grupos
5. SELECT               → Projeta colunas e expressões
6. DISTINCT             → Remove duplicatas
7. ORDER BY             → Ordena o resultado
8. LIMIT / OFFSET       → Limita as linhas retornadas
```

Entender essa ordem explica, por exemplo, por que não se pode usar alias do `SELECT` no `WHERE`.

### Convenções

```sql
-- Comentário de linha
/* Comentário
   de bloco */

-- SQL é case-insensitive para palavras-chave, mas case-sensitive para dados
-- Convenção comum: palavras-chave em UPPER CASE, nomes em lower_case
SELECT nome, email FROM clientes WHERE ativo = true;
```

---

## <a id="select">2. SELECT — Consultas</a>

### Básico

```sql
-- Todas as colunas
SELECT * FROM clientes;

-- Colunas específicas
SELECT id, nome, email FROM clientes;

-- Alias de coluna
SELECT nome AS cliente_nome, email AS contato FROM clientes;

-- Alias de tabela
SELECT c.nome, c.email FROM clientes c;

-- Expressões calculadas
SELECT nome, preco, quantidade, preco * quantidade AS total FROM itens;

-- Valores literais e funções
SELECT 'texto literal' AS coluna, NOW() AS agora, 2 + 3 AS soma;
```

### DISTINCT

```sql
-- Remover duplicatas
SELECT DISTINCT cidade FROM clientes;

-- Distinct em múltiplas colunas
SELECT DISTINCT cidade, estado FROM clientes;

-- DISTINCT ON (PostgreSQL) — primeira linha de cada grupo
SELECT DISTINCT ON (cidade) cidade, nome, saldo
FROM clientes
ORDER BY cidade, saldo DESC;
```

### Alias com expressões complexas

```sql
SELECT
    nome,
    preco * quantidade AS subtotal,
    preco * quantidade * 0.9 AS com_desconto,
    CASE
        WHEN preco * quantidade > 1000 THEN 'grande'
        ELSE 'pequeno'
    END AS porte
FROM itens_pedido;
```

---

## <a id="where">3. WHERE — Filtragem</a>

### Operadores de comparação

```sql
-- Igualdade e desigualdade
SELECT * FROM produtos WHERE preco = 29.90;
SELECT * FROM produtos WHERE preco != 29.90;   -- ou <>

-- Maior, menor, entre
SELECT * FROM produtos WHERE preco > 100;
SELECT * FROM produtos WHERE preco >= 50 AND preco <= 200;
SELECT * FROM produtos WHERE preco BETWEEN 50 AND 200;  -- inclusivo
```

### Operadores lógicos

```sql
-- AND, OR, NOT
SELECT * FROM clientes WHERE cidade = 'São Paulo' AND ativo = true;
SELECT * FROM clientes WHERE cidade = 'São Paulo' OR cidade = 'Rio de Janeiro';
SELECT * FROM clientes WHERE NOT ativo;

-- Precedência: NOT > AND > OR. Use parênteses para clareza:
SELECT * FROM produtos
WHERE (categoria = 'eletrônicos' OR categoria = 'informática')
  AND preco < 500;
```

### IN e NOT IN

```sql
SELECT * FROM clientes WHERE estado IN ('SP', 'RJ', 'MG');
SELECT * FROM clientes WHERE estado NOT IN ('AC', 'RR', 'AP');

-- IN com subquery
SELECT * FROM clientes
WHERE id IN (SELECT cliente_id FROM pedidos WHERE valor > 1000);
```

### LIKE e ILIKE

```sql
-- % = qualquer sequência de caracteres
-- _ = exatamente um caractere
SELECT * FROM clientes WHERE nome LIKE 'Ana%';       -- começa com Ana
SELECT * FROM clientes WHERE email LIKE '%@gmail.com'; -- termina com @gmail.com
SELECT * FROM clientes WHERE nome LIKE '_na%';       -- segundo e terceiro char = "na"

-- ILIKE = case-insensitive (PostgreSQL)
SELECT * FROM clientes WHERE nome ILIKE '%silva%';
```

### IS NULL / IS NOT NULL

```sql
SELECT * FROM clientes WHERE telefone IS NULL;
SELECT * FROM clientes WHERE telefone IS NOT NULL;

-- Cuidado: NULL não é comparável com = ou !=
-- Isto NÃO funciona como esperado:
-- SELECT * FROM clientes WHERE telefone = NULL;  ← sempre vazio
```

### EXISTS

```sql
-- Clientes que têm pelo menos um pedido
SELECT * FROM clientes c
WHERE EXISTS (
    SELECT 1 FROM pedidos p WHERE p.cliente_id = c.id
);

-- Clientes sem nenhum pedido
SELECT * FROM clientes c
WHERE NOT EXISTS (
    SELECT 1 FROM pedidos p WHERE p.cliente_id = c.id
);
```

### ANY e ALL

```sql
-- ANY: verdadeiro se qualquer valor satisfaz
SELECT * FROM produtos WHERE preco > ANY (SELECT preco FROM produtos WHERE categoria = 'importados');

-- ALL: verdadeiro se todos os valores satisfazem
SELECT * FROM produtos WHERE preco > ALL (SELECT preco FROM produtos WHERE categoria = 'nacional');
```

---

## <a id="order-limit">4. ORDER BY, LIMIT e OFFSET</a>

```sql
-- Ordenar ascendente (padrão)
SELECT * FROM clientes ORDER BY nome;
SELECT * FROM clientes ORDER BY nome ASC;

-- Ordenar descendente
SELECT * FROM clientes ORDER BY saldo DESC;

-- Múltiplas colunas
SELECT * FROM clientes ORDER BY cidade ASC, saldo DESC;

-- Ordenar por posição da coluna
SELECT nome, email, saldo FROM clientes ORDER BY 3 DESC;

-- NULLs: controlar posição
SELECT * FROM clientes ORDER BY telefone NULLS LAST;
SELECT * FROM clientes ORDER BY telefone NULLS FIRST;

-- LIMIT e OFFSET
SELECT * FROM produtos ORDER BY preco DESC LIMIT 10;          -- top 10
SELECT * FROM produtos ORDER BY preco DESC LIMIT 10 OFFSET 20; -- página 3

-- Sintaxe padrão SQL (FETCH)
SELECT * FROM produtos ORDER BY preco DESC
FETCH FIRST 10 ROWS ONLY;

SELECT * FROM produtos ORDER BY preco DESC
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

> **Atenção:** paginação com `OFFSET` grande é ineficiente. Para grandes datasets, prefira **keyset pagination** (cursor-based):

```sql
-- Keyset pagination (mais eficiente)
SELECT * FROM produtos
WHERE id > 1000   -- último id da página anterior
ORDER BY id
LIMIT 10;
```

---

## <a id="joins">5. JOIN — Combinação de tabelas</a>

### INNER JOIN

Retorna apenas linhas com correspondência em **ambas** as tabelas:

```sql
SELECT c.nome, p.id AS pedido_id, p.valor_total
FROM clientes c
INNER JOIN pedidos p ON p.cliente_id = c.id;
```

### LEFT JOIN (LEFT OUTER JOIN)

Retorna **todas** as linhas da tabela da esquerda, com `NULL` onde não houver correspondência:

```sql
SELECT c.nome, p.id AS pedido_id, COALESCE(p.valor_total, 0) AS valor
FROM clientes c
LEFT JOIN pedidos p ON p.cliente_id = c.id;
```

### RIGHT JOIN (RIGHT OUTER JOIN)

Inverso do LEFT — todas as linhas da tabela da direita:

```sql
SELECT c.nome, p.id AS pedido_id
FROM clientes c
RIGHT JOIN pedidos p ON p.cliente_id = c.id;
```

### FULL OUTER JOIN

Todas as linhas de **ambas** as tabelas, com `NULL` onde não houver correspondência:

```sql
SELECT a.id, a.valor AS valor_a, b.id, b.valor AS valor_b
FROM tabela_a a
FULL OUTER JOIN tabela_b b ON a.chave = b.chave;
```

### CROSS JOIN

Produto cartesiano — cada linha de uma tabela com cada linha da outra:

```sql
SELECT cores.nome, tamanhos.sigla
FROM cores
CROSS JOIN tamanhos;

-- Equivalente:
SELECT cores.nome, tamanhos.sigla
FROM cores, tamanhos;
```

### SELF JOIN

Uma tabela consigo mesma:

```sql
-- Hierarquia de funcionários
SELECT
    f.nome AS funcionario,
    g.nome AS gerente
FROM funcionarios f
LEFT JOIN funcionarios g ON f.gerente_id = g.id;
```

### NATURAL JOIN

Join automático por colunas de mesmo nome (evitar em produção — frágil):

```sql
SELECT * FROM clientes NATURAL JOIN pedidos;
```

### JOIN com múltiplas condições

```sql
SELECT *
FROM pedidos p
INNER JOIN itens i ON i.pedido_id = p.id AND i.ativo = true
INNER JOIN produtos pr ON pr.id = i.produto_id;
```

### LATERAL JOIN (PostgreSQL)

Permite subquery correlacionada como tabela — cada linha da esquerda alimenta a subquery:

```sql
-- Últimos 3 pedidos de cada cliente
SELECT c.nome, ult.id, ult.valor_total, ult.criado_em
FROM clientes c
CROSS JOIN LATERAL (
    SELECT id, valor_total, criado_em
    FROM pedidos
    WHERE cliente_id = c.id
    ORDER BY criado_em DESC
    LIMIT 3
) ult;
```

### Anti-join (encontrar linhas sem correspondência)

```sql
-- Método 1: LEFT JOIN + IS NULL
SELECT c.*
FROM clientes c
LEFT JOIN pedidos p ON p.cliente_id = c.id
WHERE p.id IS NULL;

-- Método 2: NOT EXISTS (geralmente melhor performance)
SELECT c.*
FROM clientes c
WHERE NOT EXISTS (SELECT 1 FROM pedidos p WHERE p.cliente_id = c.id);

-- Método 3: NOT IN (cuidado com NULLs)
SELECT * FROM clientes
WHERE id NOT IN (SELECT cliente_id FROM pedidos WHERE cliente_id IS NOT NULL);
```

---

## <a id="agregacoes">6. Agregações e GROUP BY</a>

### Funções de agregação

| Função | Descrição |
|---|---|
| `COUNT(*)` | Total de linhas |
| `COUNT(coluna)` | Total de valores não-NULL |
| `COUNT(DISTINCT coluna)` | Total de valores distintos |
| `SUM(coluna)` | Soma |
| `AVG(coluna)` | Média |
| `MIN(coluna)` | Menor valor |
| `MAX(coluna)` | Maior valor |
| `ARRAY_AGG(coluna)` | Agrega em array (PostgreSQL) |
| `STRING_AGG(coluna, sep)` | Concatena com separador (PostgreSQL) |
| `JSONB_AGG(expressão)` | Agrega em array JSONB (PostgreSQL) |
| `BOOL_AND(coluna)` | TRUE se todos forem TRUE |
| `BOOL_OR(coluna)` | TRUE se qualquer um for TRUE |

### GROUP BY

```sql
-- Agrupar por uma coluna
SELECT cidade, COUNT(*) AS total
FROM clientes
GROUP BY cidade
ORDER BY total DESC;

-- Agrupar por múltiplas colunas
SELECT estado, cidade, COUNT(*) AS total
FROM clientes
GROUP BY estado, cidade;

-- Agregação com expressão
SELECT
    date_trunc('month', criado_em) AS mes,
    COUNT(*) AS total_pedidos,
    SUM(valor_total) AS receita,
    AVG(valor_total) AS ticket_medio
FROM pedidos
GROUP BY date_trunc('month', criado_em)
ORDER BY mes;
```

### FILTER (PostgreSQL)

Permite filtrar dentro de cada agregação:

```sql
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE status = 'pago') AS pagos,
    COUNT(*) FILTER (WHERE status = 'pendente') AS pendentes,
    SUM(valor_total) FILTER (WHERE status = 'pago') AS receita_confirmada
FROM pedidos;
```

### GROUPING SETS, ROLLUP e CUBE

```sql
-- GROUPING SETS: múltiplos agrupamentos em uma query
SELECT estado, cidade, COUNT(*)
FROM clientes
GROUP BY GROUPING SETS (
    (estado, cidade),
    (estado),
    ()              -- total geral
);

-- ROLLUP: subtotais hierárquicos
SELECT estado, cidade, COUNT(*)
FROM clientes
GROUP BY ROLLUP (estado, cidade);
-- Gera: (estado, cidade), (estado), ()

-- CUBE: todas as combinações possíveis
SELECT estado, cidade, COUNT(*)
FROM clientes
GROUP BY CUBE (estado, cidade);
-- Gera: (estado, cidade), (estado), (cidade), ()
```

### STRING_AGG e ARRAY_AGG

```sql
-- Concatenar nomes separados por vírgula
SELECT cidade, STRING_AGG(nome, ', ' ORDER BY nome) AS clientes
FROM clientes
GROUP BY cidade;

-- Agregar em array
SELECT cliente_id, ARRAY_AGG(produto ORDER BY produto) AS produtos
FROM itens_pedido
GROUP BY cliente_id;
```

---

## <a id="having">7. HAVING</a>

`HAVING` filtra **grupos** (pós-agregação), enquanto `WHERE` filtra **linhas** (pré-agregação):

```sql
-- Cidades com mais de 100 clientes
SELECT cidade, COUNT(*) AS total
FROM clientes
GROUP BY cidade
HAVING COUNT(*) > 100
ORDER BY total DESC;

-- Meses com receita acima de 50.000
SELECT
    date_trunc('month', criado_em) AS mes,
    SUM(valor_total) AS receita
FROM pedidos
GROUP BY 1
HAVING SUM(valor_total) > 50000
ORDER BY mes;

-- HAVING com múltiplas condições
SELECT categoria, COUNT(*) AS total, AVG(preco) AS preco_medio
FROM produtos
GROUP BY categoria
HAVING COUNT(*) >= 10 AND AVG(preco) > 50;
```

---

## <a id="subqueries">8. Subqueries</a>

### Subquery escalar (retorna um valor)

```sql
SELECT nome, saldo,
       (SELECT AVG(saldo) FROM clientes) AS media_geral,
       saldo - (SELECT AVG(saldo) FROM clientes) AS diff_media
FROM clientes;
```

### Subquery em FROM (tabela derivada)

```sql
SELECT resumo.cidade, resumo.total
FROM (
    SELECT cidade, COUNT(*) AS total
    FROM clientes
    GROUP BY cidade
) resumo
WHERE resumo.total > 50;
```

### Subquery em WHERE

```sql
-- Produtos acima do preço médio
SELECT * FROM produtos
WHERE preco > (SELECT AVG(preco) FROM produtos);

-- Clientes com pelo menos um pedido grande
SELECT * FROM clientes
WHERE id IN (
    SELECT DISTINCT cliente_id FROM pedidos WHERE valor_total > 5000
);
```

### Subqueries correlacionadas

A subquery referencia a query externa — executada uma vez por linha:

```sql
-- Para cada cliente, contar pedidos
SELECT
    c.nome,
    (SELECT COUNT(*) FROM pedidos p WHERE p.cliente_id = c.id) AS total_pedidos
FROM clientes c;

-- Clientes cujo saldo é maior que a média da sua cidade
SELECT * FROM clientes c
WHERE saldo > (
    SELECT AVG(saldo) FROM clientes WHERE cidade = c.cidade
);
```

---

## <a id="ctes">9. CTEs (Common Table Expressions)</a>

CTEs (`WITH`) tornam queries complexas mais legíveis e reutilizáveis.

### CTE simples

```sql
WITH clientes_vip AS (
    SELECT id, nome, saldo
    FROM clientes
    WHERE saldo > 10000
)
SELECT cv.nome, COUNT(p.id) AS total_pedidos
FROM clientes_vip cv
LEFT JOIN pedidos p ON p.cliente_id = cv.id
GROUP BY cv.nome
ORDER BY total_pedidos DESC;
```

### Múltiplas CTEs

```sql
WITH
    vendas_mes AS (
        SELECT
            cliente_id,
            SUM(valor_total) AS total_gasto
        FROM pedidos
        WHERE criado_em >= date_trunc('month', CURRENT_DATE)
        GROUP BY cliente_id
    ),
    ranking AS (
        SELECT
            cliente_id,
            total_gasto,
            RANK() OVER (ORDER BY total_gasto DESC) AS posicao
        FROM vendas_mes
    )
SELECT c.nome, r.total_gasto, r.posicao
FROM ranking r
JOIN clientes c ON c.id = r.cliente_id
WHERE r.posicao <= 10;
```

### CTE recursiva

Ideal para hierarquias, grafos e sequências:

```sql
-- Hierarquia de categorias (árvore)
WITH RECURSIVE arvore AS (
    -- Caso base: raízes (sem pai)
    SELECT id, nome, pai_id, 1 AS nivel, nome::TEXT AS caminho
    FROM categorias
    WHERE pai_id IS NULL

    UNION ALL

    -- Caso recursivo: filhas
    SELECT c.id, c.nome, c.pai_id, a.nivel + 1, a.caminho || ' > ' || c.nome
    FROM categorias c
    INNER JOIN arvore a ON c.pai_id = a.id
)
SELECT * FROM arvore ORDER BY caminho;

-- Gerar série de datas
WITH RECURSIVE datas AS (
    SELECT DATE '2025-01-01' AS dia
    UNION ALL
    SELECT dia + 1 FROM datas WHERE dia < '2025-12-31'
)
SELECT dia FROM datas;

-- Sequência de Fibonacci
WITH RECURSIVE fib AS (
    SELECT 1 AS n, 1::BIGINT AS valor, 1::BIGINT AS anterior
    UNION ALL
    SELECT n + 1, valor + anterior, valor FROM fib WHERE n < 20
)
SELECT n, valor FROM fib;
```

### CTE com DML (writeable CTEs — PostgreSQL)

```sql
-- Deletar e mover registros em uma operação
WITH removidos AS (
    DELETE FROM sessoes
    WHERE expira_em < NOW()
    RETURNING *
)
INSERT INTO sessoes_expiradas
SELECT * FROM removidos;

-- Atualizar e retornar afetados
WITH atualizados AS (
    UPDATE clientes
    SET ativo = false
    WHERE ultimo_login < NOW() - INTERVAL '2 years'
    RETURNING id, nome, email
)
SELECT COUNT(*) AS total_desativados FROM atualizados;
```

---

## <a id="window-functions">10. Window Functions</a>

Executam cálculos sobre um conjunto de linhas **sem agrupar** o resultado.

### ROW_NUMBER, RANK, DENSE_RANK

```sql
SELECT
    nome, cidade, saldo,
    ROW_NUMBER() OVER (ORDER BY saldo DESC) AS posicao,
    RANK()       OVER (ORDER BY saldo DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY saldo DESC) AS dense_rank
FROM clientes;
```

| Função | Empates |
|---|---|
| `ROW_NUMBER` | Sem repetição (1, 2, 3, 4) |
| `RANK` | Pula posições (1, 2, 2, 4) |
| `DENSE_RANK` | Não pula posições (1, 2, 2, 3) |

### PARTITION BY

Divide o resultado em partições independentes:

```sql
-- Ranking por cidade
SELECT
    nome, cidade, saldo,
    ROW_NUMBER() OVER (PARTITION BY cidade ORDER BY saldo DESC) AS rank_na_cidade,
    SUM(saldo)   OVER (PARTITION BY cidade) AS total_cidade,
    AVG(saldo)   OVER (PARTITION BY cidade) AS media_cidade,
    COUNT(*)     OVER (PARTITION BY cidade) AS clientes_na_cidade
FROM clientes;
```

### LAG e LEAD

Acessam valores da linha anterior/próxima:

```sql
SELECT
    data,
    receita,
    LAG(receita, 1)  OVER (ORDER BY data) AS receita_dia_anterior,
    LEAD(receita, 1) OVER (ORDER BY data) AS receita_dia_seguinte,
    receita - LAG(receita) OVER (ORDER BY data) AS variacao_diaria,
    ROUND(100.0 * (receita - LAG(receita) OVER (ORDER BY data))
        / NULLIF(LAG(receita) OVER (ORDER BY data), 0), 2) AS variacao_pct
FROM vendas_diarias;
```

### Running total e Moving average

```sql
SELECT
    data,
    receita,
    SUM(receita) OVER (ORDER BY data) AS acumulado,
    AVG(receita) OVER (
        ORDER BY data
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS media_movel_7d
FROM vendas_diarias;
```

### FIRST_VALUE, LAST_VALUE, NTH_VALUE

```sql
SELECT
    nome, saldo,
    FIRST_VALUE(nome) OVER (ORDER BY saldo DESC) AS mais_rico,
    LAST_VALUE(nome) OVER (
        ORDER BY saldo DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS menos_rico,
    NTILE(4) OVER (ORDER BY saldo) AS quartil
FROM clientes;
```

### Frame specification (ROWS / RANGE)

```sql
-- ROWS: linhas físicas
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW        -- 3 linhas
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW -- desde o início

-- RANGE: valores lógicos
RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW

-- GROUPS (PostgreSQL 11+): grupos de pares
GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING
```

### Named window

Reutilizar definição de janela:

```sql
SELECT
    nome, cidade, saldo,
    ROW_NUMBER() OVER w AS pos,
    SUM(saldo)   OVER w AS acumulado,
    AVG(saldo)   OVER w AS media_acumulada
FROM clientes
WINDOW w AS (PARTITION BY cidade ORDER BY saldo DESC);
```

---

## <a id="set-operations">11. UNION, INTERSECT e EXCEPT</a>

### UNION

Combina resultados de duas queries (remove duplicatas):

```sql
SELECT nome, email FROM clientes_br
UNION
SELECT nome, email FROM clientes_us;
```

### UNION ALL

Mantém duplicatas (mais rápido):

```sql
SELECT nome FROM clientes_br
UNION ALL
SELECT nome FROM clientes_us;
```

### INTERSECT

Linhas presentes em **ambas** as queries:

```sql
SELECT email FROM clientes
INTERSECT
SELECT email FROM newsletter_assinantes;
```

### EXCEPT

Linhas presentes na primeira mas **não** na segunda:

```sql
SELECT email FROM clientes
EXCEPT
SELECT email FROM blacklist;
```

> Todas as set operations exigem **mesmo número e tipos compatíveis** de colunas.

---

## <a id="insert">12. INSERT</a>

### Inserção básica

```sql
INSERT INTO clientes (nome, email, ativo)
VALUES ('Ana Silva', 'ana@exemplo.com', true);
```

### Múltiplas linhas

```sql
INSERT INTO clientes (nome, email) VALUES
    ('Carlos', 'carlos@email.com'),
    ('Maria', 'maria@email.com'),
    ('João', 'joao@email.com');
```

### INSERT com RETURNING (PostgreSQL)

```sql
INSERT INTO clientes (nome, email)
VALUES ('Pedro', 'pedro@email.com')
RETURNING id, criado_em;
```

### INSERT ... SELECT

```sql
INSERT INTO clientes_ativos (id, nome, email)
SELECT id, nome, email
FROM clientes
WHERE ativo = true;
```

### INSERT com CTE

```sql
WITH novos AS (
    SELECT nome, email
    FROM staging_importacao
    WHERE validado = true
)
INSERT INTO clientes (nome, email)
SELECT nome, email FROM novos;
```

### COPY / \\copy (carga em massa — PostgreSQL)

```sql
-- Servidor: COPY (requer acesso ao filesystem do servidor)
COPY clientes (nome, email) FROM '/tmp/clientes.csv' WITH (FORMAT csv, HEADER true);

-- Cliente psql: \copy (arquivo local)
\copy clientes(nome, email) FROM 'clientes.csv' WITH CSV HEADER DELIMITER ','

-- Exportar
COPY (SELECT * FROM clientes WHERE ativo) TO '/tmp/ativos.csv' WITH (FORMAT csv, HEADER true);
```

---

## <a id="update">13. UPDATE</a>

### Básico

```sql
UPDATE clientes
SET ativo = false
WHERE ultimo_login < '2024-01-01';
```

### Múltiplas colunas

```sql
UPDATE clientes
SET nome = 'Ana Maria', atualizado_em = NOW()
WHERE id = 1;
```

### UPDATE com RETURNING (PostgreSQL)

```sql
UPDATE produtos
SET preco = preco * 1.10
WHERE categoria = 'importados'
RETURNING id, nome, preco;
```

### UPDATE com JOIN (FROM — PostgreSQL)

```sql
UPDATE pedidos p
SET status = 'cancelado'
FROM clientes c
WHERE p.cliente_id = c.id
  AND c.ativo = false;
```

### UPDATE com subquery

```sql
UPDATE produtos
SET preco = preco * 0.9
WHERE id IN (
    SELECT produto_id FROM estoque WHERE quantidade > 1000
);
```

### UPDATE com CTE

```sql
WITH inativos AS (
    SELECT id FROM clientes WHERE ultimo_login < NOW() - INTERVAL '1 year'
)
UPDATE clientes SET ativo = false
WHERE id IN (SELECT id FROM inativos);
```

> **Dica:** antes de executar um `UPDATE`, teste o `WHERE` com `SELECT` para confirmar as linhas afetadas.

---

## <a id="delete">14. DELETE e TRUNCATE</a>

### DELETE básico

```sql
DELETE FROM clientes WHERE ativo = false;
```

### DELETE com RETURNING (PostgreSQL)

```sql
DELETE FROM sessoes
WHERE expira_em < NOW()
RETURNING id, usuario_id;
```

### DELETE com JOIN (USING — PostgreSQL)

```sql
DELETE FROM itens_pedido i
USING pedidos p
WHERE i.pedido_id = p.id
  AND p.status = 'cancelado';
```

### DELETE em cascata

Se a foreign key foi definida com `ON DELETE CASCADE`, o delete do pai remove os filhos automaticamente.

### TRUNCATE

Remove **todas** as linhas — muito mais rápido que `DELETE` sem `WHERE`:

```sql
TRUNCATE TABLE logs_temp;

-- Múltiplas tabelas com cascata
TRUNCATE TABLE pedidos, itens_pedido CASCADE;

-- Reiniciar sequences
TRUNCATE TABLE clientes RESTART IDENTITY CASCADE;
```

| | DELETE | TRUNCATE |
|---|---|---|
| Filtragem | Suporta WHERE | Não |
| Triggers | Executa row triggers | Não (apenas statement-level) |
| MVCC | Marca linhas como mortas | Remove páginas inteiras |
| Velocidade | Lento para muitas linhas | Muito rápido |
| Rollback | Sim | Sim (em PostgreSQL) |

---

## <a id="upsert">15. UPSERT (INSERT ON CONFLICT / MERGE)</a>

### INSERT ... ON CONFLICT (PostgreSQL)

```sql
-- Inserir ou atualizar (upsert)
INSERT INTO clientes (nome, email, telefone)
VALUES ('Ana', 'ana@email.com', '11999999999')
ON CONFLICT (email)
DO UPDATE SET
    nome = EXCLUDED.nome,
    telefone = EXCLUDED.telefone,
    atualizado_em = NOW();

-- Inserir ou ignorar duplicata
INSERT INTO clientes (nome, email)
VALUES ('Ana', 'ana@email.com')
ON CONFLICT (email) DO NOTHING;

-- Com RETURNING
INSERT INTO config (chave, valor)
VALUES ('max_tentativas', '5')
ON CONFLICT (chave)
DO UPDATE SET valor = EXCLUDED.valor
RETURNING *;
```

`EXCLUDED` refere-se à linha que **tentou** ser inserida.

### MERGE (SQL:2016 — PostgreSQL 15+)

```sql
MERGE INTO clientes AS alvo
USING staging_clientes AS fonte
ON alvo.email = fonte.email
WHEN MATCHED THEN
    UPDATE SET
        nome = fonte.nome,
        telefone = fonte.telefone,
        atualizado_em = NOW()
WHEN NOT MATCHED THEN
    INSERT (nome, email, telefone)
    VALUES (fonte.nome, fonte.email, fonte.telefone);
```

---

## <a id="ddl">16. DDL — CREATE, ALTER, DROP</a>

### CREATE TABLE

```sql
CREATE TABLE clientes (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
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
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cliente_id  BIGINT NOT NULL REFERENCES clientes(id) ON DELETE CASCADE,
    valor_total NUMERIC(12,2) NOT NULL CHECK (valor_total >= 0),
    status      VARCHAR(20) DEFAULT 'pendente',
    criado_em   TIMESTAMPTZ DEFAULT NOW()
);
```

### ALTER TABLE

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

-- Adicionar NOT NULL
ALTER TABLE clientes ALTER COLUMN nome SET NOT NULL;

-- Remover NOT NULL
ALTER TABLE clientes ALTER COLUMN nome DROP NOT NULL;

-- Renomear tabela
ALTER TABLE clientes RENAME TO customers;
```

### DROP

```sql
DROP TABLE IF EXISTS clientes;
DROP TABLE clientes CASCADE;  -- remove dependentes (FKs, views, etc.)
```

### CREATE TABLE AS (a partir de query)

```sql
CREATE TABLE clientes_sp AS
SELECT * FROM clientes WHERE estado = 'SP';
```

### Tabelas temporárias

```sql
-- Existe apenas durante a sessão
CREATE TEMP TABLE tmp_importacao (
    linha INT,
    dados TEXT
);

-- Removida ao final da transação
CREATE TEMP TABLE tmp_calc (...) ON COMMIT DROP;
```

### Tabelas UNLOGGED (PostgreSQL)

Sem WAL — escrita mais rápida, mas **não sobrevivem a crash**:

```sql
CREATE UNLOGGED TABLE cache_sessoes (
    token TEXT PRIMARY KEY,
    dados JSONB
);
```

---

## <a id="tipos-dados">17. Tipos de Dados</a>

### Numéricos

| Tipo | Tamanho | Uso |
|---|---|---|
| `SMALLINT` | 2 bytes | Inteiros pequenos (-32.768 a 32.767) |
| `INTEGER` (ou `INT`) | 4 bytes | Uso geral |
| `BIGINT` | 8 bytes | Inteiros grandes |
| `NUMERIC(p,s)` | variável | Precisão exata (financeiro) |
| `REAL` | 4 bytes | Ponto flutuante (6 dígitos) |
| `DOUBLE PRECISION` | 8 bytes | Ponto flutuante (15 dígitos) |
| `SERIAL` / `BIGSERIAL` | 4/8 bytes | Auto-incremento (legado — prefer `GENERATED AS IDENTITY`) |

### Texto

| Tipo | Descrição |
|---|---|
| `VARCHAR(n)` | Texto com limite de `n` caracteres |
| `CHAR(n)` | Tamanho fixo (preenchido com espaços) |
| `TEXT` | Sem limite (PostgreSQL: mesma performance que `VARCHAR`) |

### Data e hora

| Tipo | Descrição | Exemplo |
|---|---|---|
| `DATE` | Apenas data | `'2025-12-31'` |
| `TIME` | Apenas hora | `'14:30:00'` |
| `TIMESTAMP` | Data + hora sem timezone | `'2025-12-31 14:30:00'` |
| `TIMESTAMPTZ` | Data + hora com timezone (recomendado) | `'2025-12-31 14:30:00-03'` |
| `INTERVAL` | Duração | `'3 days 4 hours'` |

```sql
SELECT NOW();                                    -- timestamp atual
SELECT CURRENT_DATE;                             -- date atual
SELECT CURRENT_DATE + INTERVAL '30 days';        -- daqui 30 dias
SELECT AGE(TIMESTAMP '2000-01-01');              -- idade
SELECT EXTRACT(YEAR FROM NOW());                 -- extrair ano
SELECT date_trunc('month', NOW());               -- truncar para início do mês
```

### Boolean

```sql
-- Valores aceitos: TRUE, FALSE, NULL, 't', 'f', 'yes', 'no', '1', '0'
CREATE TABLE config (ativo BOOLEAN DEFAULT TRUE);
```

### UUID

```sql
-- PostgreSQL 13+: nativo
SELECT gen_random_uuid();

-- Com extensão
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();

CREATE TABLE sessoes (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    usuario_id INTEGER
);
```

### Arrays (PostgreSQL)

```sql
CREATE TABLE produtos (nome TEXT, tags TEXT[]);

INSERT INTO produtos VALUES ('Notebook', ARRAY['eletrônico', 'informática']);
INSERT INTO produtos VALUES ('Caderno', '{"papelaria","escola"}');

SELECT * FROM produtos WHERE 'eletrônico' = ANY(tags);
SELECT * FROM produtos WHERE tags @> ARRAY['informática'];
SELECT unnest(tags) FROM produtos;
```

### Outros tipos relevantes (PostgreSQL)

| Tipo | Uso |
|---|---|
| `JSONB` / `JSON` | Dados semi-estruturados |
| `BYTEA` | Dados binários |
| `INET` / `CIDR` | Endereços IP / redes |
| `TSVECTOR` / `TSQUERY` | Full-text search |
| `DATERANGE`, `INT4RANGE` | Intervalos (ranges) |
| `ENUM` | Tipos enumerados |

---

## <a id="constraints">18. Constraints</a>

### PRIMARY KEY

```sql
CREATE TABLE clientes (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);

-- Composta
CREATE TABLE itens (
    pedido_id BIGINT,
    produto_id BIGINT,
    PRIMARY KEY (pedido_id, produto_id)
);
```

### UNIQUE

```sql
ALTER TABLE clientes ADD CONSTRAINT uk_email UNIQUE (email);

-- Unique composto
ALTER TABLE matriculas ADD CONSTRAINT uk_aluno_turma UNIQUE (aluno_id, turma_id);
```

### FOREIGN KEY

```sql
ALTER TABLE pedidos ADD CONSTRAINT fk_cliente
    FOREIGN KEY (cliente_id) REFERENCES clientes(id)
    ON DELETE CASCADE
    ON UPDATE CASCADE;
```

Ações de referência:

| Ação | Comportamento |
|---|---|
| `CASCADE` | Propaga delete/update |
| `SET NULL` | Define como NULL |
| `SET DEFAULT` | Define como valor default |
| `RESTRICT` | Bloqueia a operação (imediato) |
| `NO ACTION` (padrão) | Bloqueia a operação (verificado ao final da transação) |

### CHECK

```sql
ALTER TABLE produtos ADD CONSTRAINT chk_preco CHECK (preco >= 0);

-- Com múltiplas condições
ALTER TABLE funcionarios ADD CONSTRAINT chk_salario
    CHECK (salario > 0 AND salario <= 999999.99);
```

### NOT NULL

```sql
ALTER TABLE clientes ALTER COLUMN nome SET NOT NULL;
```

### EXCLUSION CONSTRAINT (PostgreSQL)

Evita sobreposição de intervalos:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE reservas (
    sala INT,
    periodo TSTZRANGE,
    EXCLUDE USING GIST (sala WITH =, periodo WITH &&)
);
```

### Constraints deferidas

```sql
-- Verificação adiada para o final da transação
ALTER TABLE pedidos ADD CONSTRAINT fk_cliente
    FOREIGN KEY (cliente_id) REFERENCES clientes(id)
    DEFERRABLE INITIALLY DEFERRED;
```

---

## <a id="indices">19. Índices</a>

### Conceito

Índices aceleram consultas ao evitar full table scans, mas adicionam custo a escrita (`INSERT`, `UPDATE`, `DELETE`).

### Tipos (PostgreSQL)

| Tipo | Quando usar |
|---|---|
| **B-tree** (padrão) | Comparações `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE 'abc%'` |
| **Hash** | Apenas igualdade `=` (raramente necessário) |
| **GIN** | Arrays, JSONB, full-text search, trigramas |
| **GiST** | Dados espaciais, ranges, full-text search |
| **BRIN** | Colunas com correlação física (timestamps em append-only) |

### Criação

```sql
-- B-tree simples
CREATE INDEX idx_clientes_email ON clientes (email);

-- Índice único
CREATE UNIQUE INDEX idx_clientes_cpf ON clientes (cpf);

-- Índice composto (ordem importa!)
CREATE INDEX idx_pedidos_cli_data ON pedidos (cliente_id, criado_em DESC);

-- Índice parcial (somente linhas que satisfazem a condição)
CREATE INDEX idx_clientes_ativos ON clientes (email) WHERE ativo = true;

-- Índice de expressão
CREATE INDEX idx_clientes_email_lower ON clientes (lower(email));

-- Covering index (index-only scan)
CREATE INDEX idx_pedidos_status ON pedidos (status) INCLUDE (valor_total, criado_em);

-- Criação concorrente (não bloqueia escrita)
CREATE INDEX CONCURRENTLY idx_pedidos_valor ON pedidos (valor_total);
```

### Manutenção

```sql
-- Remover
DROP INDEX IF EXISTS idx_clientes_email;
DROP INDEX CONCURRENTLY idx_clientes_email;

-- Reindexar
REINDEX INDEX idx_clientes_email;
REINDEX TABLE clientes;
REINDEX (VERBOSE) TABLE CONCURRENTLY clientes;  -- PostgreSQL 12+
```

### Quando criar índices

- Colunas usadas frequentemente em `WHERE`, `JOIN`, `ORDER BY`.
- Colunas com alta seletividade (muitos valores distintos).
- **Não** crie em colunas pouco seletivas (ex: boolean com 90% TRUE).
- **Não** crie em excesso — cada índice custa a cada escrita.

### Verificar uso

```sql
-- Índices não utilizados
SELECT schemaname, relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS tamanho
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## <a id="views">20. Views e Materialized Views</a>

### Views (consultas reutilizáveis)

```sql
CREATE VIEW vw_pedidos_pagos AS
SELECT p.id, c.nome AS cliente, p.valor_total, p.criado_em
FROM pedidos p
JOIN clientes c ON c.id = p.cliente_id
WHERE p.status = 'pago';

-- Consultar
SELECT * FROM vw_pedidos_pagos WHERE valor_total > 1000;

-- Substituir
CREATE OR REPLACE VIEW vw_pedidos_pagos AS
SELECT p.id, c.nome AS cliente, p.valor_total, p.criado_em, c.email
FROM pedidos p
JOIN clientes c ON c.id = p.cliente_id
WHERE p.status = 'pago';

-- Remover
DROP VIEW IF EXISTS vw_pedidos_pagos;
```

Views são **virtual** — a query é executada a cada acesso.

### Views atualizáveis

Views simples sobre uma tabela são automaticamente atualizáveis:

```sql
CREATE VIEW vw_clientes_sp AS
SELECT * FROM clientes WHERE estado = 'SP'
WITH CHECK OPTION;  -- impede inserir linhas que não satisfazem o filtro

UPDATE vw_clientes_sp SET saldo = 5000 WHERE id = 1;
```

### Materialized Views (PostgreSQL)

Armazenam o resultado em disco — consulta rápida, mas dados podem ficar desatualizados:

```sql
CREATE MATERIALIZED VIEW mv_vendas_mensal AS
SELECT
    date_trunc('month', criado_em) AS mes,
    COUNT(*) AS total_pedidos,
    SUM(valor_total) AS receita
FROM pedidos
GROUP BY 1;

-- Consultar (instantâneo)
SELECT * FROM mv_vendas_mensal;

-- Atualizar dados
REFRESH MATERIALIZED VIEW mv_vendas_mensal;

-- Refresh concorrente (sem bloquear leitura — requer índice UNIQUE)
CREATE UNIQUE INDEX idx_mv_vendas_mes ON mv_vendas_mensal (mes);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_vendas_mensal;
```

---

## <a id="transacoes">21. Transações</a>

### Conceito

Transações agrupam operações em uma unidade atômica — ou tudo acontece, ou nada.

**Propriedades ACID:**

| Propriedade | Significado |
|---|---|
| **Atomicity** | Tudo ou nada |
| **Consistency** | Dados sempre em estado válido |
| **Isolation** | Transações não interferem entre si |
| **Durability** | Dados persistem após commit |

### Uso básico

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

    -- Se erro:
    ROLLBACK TO SAVEPOINT sp1;

    -- Continuar
    INSERT INTO logs (...) VALUES (...);
COMMIT;
```

### Níveis de isolamento

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;     -- padrão no PostgreSQL
BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE;
```

| Nível | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Committed (padrão) | Não | Possível | Possível |
| Repeatable Read | Não | Não | Não* |
| Serializable | Não | Não | Não |

(*) PostgreSQL previne phantom reads em Repeatable Read via SSI.

### Locks

```sql
-- Lock de linha
SELECT * FROM contas WHERE id = 1 FOR UPDATE;

-- Lock com NOWAIT (erro se já estiver locked)
SELECT * FROM contas WHERE id = 1 FOR UPDATE NOWAIT;

-- SKIP LOCKED (ignora locked — útil para filas de trabalho)
SELECT * FROM tarefas
WHERE status = 'pendente'
FOR UPDATE SKIP LOCKED
LIMIT 1;

-- Lock compartilhado (permite leitura, bloqueia escrita)
SELECT * FROM clientes WHERE id = 1 FOR SHARE;
```

### Timeouts

```sql
SET lock_timeout = '10s';                               -- espera de lock
SET statement_timeout = '60s';                          -- duração máxima de query
SET idle_in_transaction_session_timeout = '5min';       -- transação ociosa
```

---

## <a id="dcl">22. DCL — Permissões (GRANT/REVOKE)</a>

### GRANT

```sql
-- Acesso ao banco
GRANT CONNECT ON DATABASE app_db TO app_user;

-- Uso de schema
GRANT USAGE ON SCHEMA vendas TO app_user;

-- Em tabelas
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA vendas TO app_user;

-- Em sequences
GRANT USAGE ON ALL SEQUENCES IN SCHEMA vendas TO app_user;

-- Executar funções
GRANT EXECUTE ON FUNCTION calcular_total(BIGINT) TO app_user;

-- Todos os privilégios
GRANT ALL PRIVILEGES ON DATABASE app_db TO admin_user;

-- Permissões para tabelas futuras
ALTER DEFAULT PRIVILEGES IN SCHEMA vendas
    GRANT SELECT, INSERT ON TABLES TO app_user;
```

### REVOKE

```sql
REVOKE INSERT ON vendas.pedidos FROM app_user;
REVOKE ALL PRIVILEGES ON DATABASE app_db FROM app_user;
```

### Roles (grupos)

```sql
-- Criar grupo
CREATE ROLE equipe_backend NOLOGIN;

-- Criar usuário
CREATE ROLE app_user WITH LOGIN PASSWORD 'senha_forte';

-- Adicionar ao grupo
GRANT equipe_backend TO app_user;

-- Conceder permissões ao grupo (todos os membros herdam)
GRANT SELECT ON ALL TABLES IN SCHEMA public TO equipe_backend;
```

### Verificar permissões

```sql
-- Via information_schema
SELECT grantee, table_schema, table_name, privilege_type
FROM information_schema.table_privileges
WHERE grantee = 'app_user';

-- Via psql
\dp schema.*
\du             -- listar roles
```

### Boas práticas

- Use **roles de grupo** em vez de conceder diretamente a usuários.
- Aplique o **princípio do menor privilégio**.
- Revise permissões periodicamente.

---

## <a id="funcoes">23. Funções e Expressões Úteis</a>

### Strings

```sql
SELECT
    LENGTH('PostgreSQL'),                          -- 10
    UPPER('texto'),                                -- TEXTO
    LOWER('TEXTO'),                                -- texto
    INITCAP('hello world'),                        -- Hello World
    TRIM('  espaços  '),                           -- 'espaços'
    LTRIM('  esquerda'),                           -- 'esquerda'
    RTRIM('direita  '),                            -- 'direita'
    REPLACE('foo bar foo', 'foo', 'baz'),          -- baz bar baz
    SUBSTRING('PostgreSQL' FROM 1 FOR 4),          -- Post
    LEFT('PostgreSQL', 4),                         -- Post
    RIGHT('PostgreSQL', 3),                        -- SQL
    CONCAT('Hello', ' ', 'World'),                 -- Hello World
    'Hello' || ' ' || 'World',                     -- Hello World
    LPAD('42', 5, '0'),                            -- 00042
    RPAD('AB', 5, '-'),                            -- AB---
    REVERSE('abc'),                                -- cba
    REPEAT('ab', 3),                               -- ababab
    SPLIT_PART('a.b.c', '.', 2),                   -- b
    POSITION('SQL' IN 'PostgreSQL'),               -- 8
    TRANSLATE('abc', 'abc', 'xyz');                 -- xyz
```

### Numéricas

```sql
SELECT
    ROUND(3.14159, 2),           -- 3.14
    CEIL(3.2),                   -- 4
    FLOOR(3.8),                  -- 3
    TRUNC(3.14159, 2),           -- 3.14 (sem arredondamento)
    ABS(-42),                    -- 42
    MOD(10, 3),                  -- 1
    POWER(2, 10),                -- 1024
    SQRT(144),                   -- 12
    GREATEST(1, 5, 3),           -- 5
    LEAST(1, 5, 3),              -- 1
    RANDOM(),                    -- 0.0 a 1.0 (PostgreSQL)
    FLOOR(RANDOM() * 100 + 1);   -- inteiro aleatório 1-100
```

### Data e hora

```sql
SELECT
    NOW(),                                              -- timestamp atual
    CURRENT_DATE,                                       -- date atual
    CURRENT_TIME,                                       -- time atual
    CURRENT_TIMESTAMP,                                  -- equivalente a NOW()
    EXTRACT(YEAR FROM NOW()),                           -- 2026
    EXTRACT(MONTH FROM NOW()),                          -- 2
    EXTRACT(DOW FROM NOW()),                            -- dia da semana (0=dom)
    date_trunc('month', NOW()),                         -- primeiro instante do mês
    date_trunc('hour', NOW()),                          -- truncar para hora
    NOW() + INTERVAL '7 days',                          -- daqui 7 dias
    NOW() - INTERVAL '3 hours',                         -- 3 horas atrás
    AGE(TIMESTAMP '2000-01-01'),                        -- idade
    AGE('2025-12-31'::date, '2025-01-01'::date),       -- diferença
    TO_CHAR(NOW(), 'DD/MM/YYYY HH24:MI:SS'),           -- formatar
    TO_DATE('31/12/2025', 'DD/MM/YYYY'),                -- parse date
    TO_TIMESTAMP('2025-12-31 14:30', 'YYYY-MM-DD HH24:MI'),
    MAKE_DATE(2025, 12, 31),                            -- construir date
    DATE_PART('epoch', NOW())::BIGINT;                  -- Unix timestamp
```

### Conversão (CAST)

```sql
SELECT
    CAST('42' AS INTEGER),                -- 42
    '42'::INTEGER,                        -- PostgreSQL shorthand
    CAST(NOW() AS DATE),                  -- timestamp → date
    42::TEXT,                             -- int → text
    TO_NUMBER('1.234,56', '9G999D99');    -- parse número localizado
```

---

## <a id="case-coalesce">24. CASE, COALESCE e NULLIF</a>

### CASE simples

```sql
SELECT
    nome,
    status,
    CASE status
        WHEN 'ativo' THEN 'Conta ativa'
        WHEN 'inativo' THEN 'Conta inativa'
        WHEN 'bloqueado' THEN 'Conta bloqueada'
        ELSE 'Desconhecido'
    END AS descricao
FROM clientes;
```

### CASE com condições (searched CASE)

```sql
SELECT
    nome,
    saldo,
    CASE
        WHEN saldo > 50000 THEN 'VIP'
        WHEN saldo > 10000 THEN 'Premium'
        WHEN saldo > 0     THEN 'Regular'
        ELSE 'Zerado'
    END AS classificacao
FROM clientes;
```

### CASE em agregações

```sql
SELECT
    COUNT(*) AS total,
    SUM(CASE WHEN status = 'pago' THEN 1 ELSE 0 END) AS pagos,
    SUM(CASE WHEN status = 'pendente' THEN 1 ELSE 0 END) AS pendentes,
    SUM(CASE WHEN status = 'pago' THEN valor_total ELSE 0 END) AS receita
FROM pedidos;
```

### CASE em ORDER BY

```sql
SELECT * FROM tarefas
ORDER BY
    CASE prioridade
        WHEN 'crítica' THEN 1
        WHEN 'alta'    THEN 2
        WHEN 'média'   THEN 3
        WHEN 'baixa'   THEN 4
        ELSE 5
    END;
```

### COALESCE

Retorna o primeiro valor não-NULL:

```sql
SELECT
    nome,
    COALESCE(telefone, email, 'sem contato') AS contato,
    COALESCE(apelido, nome) AS nome_exibicao
FROM clientes;

-- Uso comum: evitar NULL em cálculos
SELECT nome, COALESCE(SUM(valor_total), 0) AS total
FROM clientes c
LEFT JOIN pedidos p ON p.cliente_id = c.id
GROUP BY nome;
```

### NULLIF

Retorna `NULL` se os dois valores forem iguais (útil para evitar divisão por zero):

```sql
-- Evitar divisão por zero
SELECT
    total_receita / NULLIF(total_pedidos, 0) AS ticket_medio
FROM resumo_mensal;

-- Converter string vazia para NULL
SELECT NULLIF(telefone, '') FROM clientes;
```

---

## <a id="regex">25. Expressões Regulares e Pattern Matching</a>

### LIKE / ILIKE

```sql
-- % = qualquer sequência, _ = exatamente um caractere
SELECT * FROM clientes WHERE nome LIKE 'Ana%';
SELECT * FROM clientes WHERE email LIKE '%@gmail.com';
SELECT * FROM clientes WHERE nome LIKE '___a%';         -- 4º char é 'a'

-- ILIKE = case-insensitive (PostgreSQL)
SELECT * FROM clientes WHERE nome ILIKE '%silva%';
```

### SIMILAR TO (SQL padrão)

```sql
SELECT * FROM clientes
WHERE nome SIMILAR TO '%(Silva|Santos|Souza)%';
```

### Operadores POSIX (PostgreSQL)

```sql
-- ~ = match (case-sensitive)
SELECT * FROM clientes WHERE email ~ '^[a-z]+@empresa\.com$';

-- ~* = match (case-insensitive)
SELECT * FROM clientes WHERE email ~* 'gmail';

-- !~ = não match
SELECT * FROM clientes WHERE email !~ '@empresa\.com$';

-- Extrair com substring
SELECT SUBSTRING(email FROM '@(.+)$') AS dominio FROM clientes;

-- regexp_matches (retorna arrays)
SELECT regexp_matches('abc 123 def 456', '\d+', 'g');

-- regexp_replace
SELECT regexp_replace('  múltiplos   espaços  ', '\s+', ' ', 'g');

-- regexp_split_to_table
SELECT regexp_split_to_table('a,b,,c', ',');

-- regexp_split_to_array
SELECT regexp_split_to_array('a,b,c', ',');
```

---

## <a id="json">26. JSON/JSONB</a>

`JSONB` (binário, mais eficiente para consultas) vs `JSON` (texto puro):

### Operações básicas

```sql
CREATE TABLE eventos (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tipo TEXT,
    dados JSONB DEFAULT '{}'::jsonb
);

INSERT INTO eventos (tipo, dados) VALUES (
    'compra',
    '{"produto": "Notebook", "valor": 4500, "tags": ["eletrônico", "premium"]}'
);
```

### Operadores de acesso

```sql
-- -> retorna JSONB
SELECT dados->'produto' FROM eventos;           -- "Notebook"

-- ->> retorna TEXT
SELECT dados->>'produto' FROM eventos;          -- Notebook

-- Acesso aninhado
SELECT dados->'cliente'->>'nome' FROM eventos;

-- #>> caminho como array
SELECT dados #>> '{cliente,nome}' FROM eventos;
```

### Operadores de consulta

```sql
-- @> contém
SELECT * FROM eventos WHERE dados @> '{"produto": "Notebook"}';

-- ? chave existe
SELECT * FROM eventos WHERE dados ? 'produto';

-- Busca em arrays JSON
SELECT * FROM eventos WHERE dados->'tags' ? 'premium';
```

### Manipulação

```sql
-- Adicionar chave
UPDATE eventos SET dados = dados || '{"prioridade": "alta"}'::jsonb;

-- Remover chave
UPDATE eventos SET dados = dados - 'prioridade';

-- jsonb_set (alterar valor aninhado)
UPDATE eventos SET dados = jsonb_set(dados, '{produto}', '"Notebook Pro"');

-- Construir JSONB
SELECT jsonb_build_object('nome', nome, 'email', email) FROM clientes;

-- Agregar em JSON
SELECT jsonb_agg(jsonb_build_object('id', id, 'nome', nome)) FROM clientes;

-- Expandir em linhas
SELECT * FROM jsonb_each_text('{"a": "1", "b": "2"}'::jsonb);

-- Expandir array
SELECT jsonb_array_elements_text('["a","b","c"]'::jsonb);
```

### Índices para JSONB

```sql
-- GIN (suporta @>, ?, ?|, ?&)
CREATE INDEX idx_eventos_dados ON eventos USING gin (dados);

-- GIN jsonb_path_ops (mais compacto, apenas @>)
CREATE INDEX idx_eventos_path ON eventos USING gin (dados jsonb_path_ops);

-- B-tree em chave específica
CREATE INDEX idx_eventos_produto ON eventos ((dados->>'produto'));
```

---

## <a id="explain">27. EXPLAIN — Análise de Planos</a>

### Uso

```sql
-- Plano estimado (não executa)
EXPLAIN SELECT * FROM clientes WHERE email = 'ana@email.com';

-- Plano real com tempos e buffers
EXPLAIN (ANALYZE, BUFFERS, TIMING)
SELECT * FROM clientes WHERE email = 'ana@email.com';

-- Formato JSON (para ferramentas visuais)
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;
```

### Interpretação

| Nó | Significado |
|---|---|
| `Seq Scan` | Full table scan (leitura sequencial) |
| `Index Scan` | Busca via índice + acesso ao heap |
| `Index Only Scan` | Dados vêm apenas do índice (ideal) |
| `Bitmap Index/Heap Scan` | Índice bitmap + acesso ao heap |
| `Nested Loop` | Join: loop aninhado |
| `Hash Join` | Join: hash em memória |
| `Merge Join` | Join: merge de listas ordenadas |
| `Sort` | Ordenação (pode usar disco se `work_mem` insuficiente) |
| `Aggregate` | Agregação (COUNT, SUM, etc.) |

### Como ler

```
Hash Join  (cost=10.50..32.65 rows=100 width=128) (actual time=0.5..1.2 rows=95 loops=1)
           ↑ custo estimado     ↑ estimativa     ↑ tamanho       ↑ tempo real  ↑ linhas reais
```

- **cost**: `custo_inicio..custo_total` (unidades arbitrárias)
- **rows**: linhas estimadas vs reais
- **actual time**: tempo em ms do primeiro ao último resultado
- **Buffers**: `shared hit` (cache) vs `shared read` (disco)

### pg_stat_statements

```sql
-- Habilitar em postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'

CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top queries mais demoradas
SELECT
    SUBSTRING(query, 1, 80) AS query,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(mean_exec_time::numeric, 2) AS media_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

---

## <a id="boas-praticas">28. Boas Práticas e Otimização</a>

### Escrita de queries

- **Evite `SELECT *`** — selecione apenas colunas necessárias.
- **Use alias** para tabelas em JOINs múltiplos.
- **Prefira `EXISTS`** sobre `IN` com subqueries (especialmente com NULLs).
- **Evite funções em colunas no WHERE** sem índice correspondente:

```sql
-- Ruim (não usa índice em email):
WHERE UPPER(email) = 'ANA@EMAIL.COM'

-- Bom (com índice em lower(email)):
WHERE lower(email) = 'ana@email.com'
```

### Índices

- Crie índices para colunas em `WHERE`, `JOIN` e `ORDER BY` frequentes.
- Prefira índices compostos na **ordem usada nas queries**.
- Use índices parciais quando filtrar subconjuntos recorrentes.
- Remova índices não utilizados (verifique `pg_stat_user_indexes`).
- Cada índice tem custo em `INSERT`/`UPDATE`/`DELETE`. Não crie em excesso.

### Performance

- Use `EXPLAIN ANALYZE` para identificar gargalos.
- Habilite `pg_stat_statements` para encontrar queries custosas.
- Prefira **keyset pagination** sobre `OFFSET` para grandes datasets.
- Use **batch operations** (multi-row INSERT, CTEs com UPDATE/DELETE).
- Considere **Materialized Views** para agregações pesadas recorrentes.
- Ajuste `work_mem` para operações de sort/hash.

### Integridade e segurança

- Use **transações** para operações que afetam múltiplas tabelas.
- Defina **constraints** (FK, CHECK, UNIQUE) para garantir integridade.
- Aplique **princípio do menor privilégio** em permissões.
- Valide dados na aplicação **e** no banco (defesa em profundidade).
- Use `statement_timeout` para evitar queries infinitas.

### Convenções de nomenclatura

| Objeto | Convenção sugerida | Exemplo |
|---|---|---|
| Tabelas | plural, snake_case | `clientes`, `itens_pedido` |
| Colunas | singular, snake_case | `nome`, `criado_em` |
| Primary Key | `id` ou `tabela_id` | `id`, `cliente_id` |
| Foreign Key | `tabela_referenciada_id` | `cliente_id` |
| Índice | `idx_tabela_colunas` | `idx_clientes_email` |
| Unique | `uk_tabela_colunas` | `uk_clientes_cpf` |
| Check | `chk_tabela_regra` | `chk_pedidos_valor` |
| View | `vw_descricao` | `vw_pedidos_pagos` |
| Materialized View | `mv_descricao` | `mv_vendas_mensal` |
| Função | verbo_substantivo | `calcular_desconto()` |
| Trigger | `trg_tabela_evento` | `trg_clientes_update` |
| Sequence | `seq_tabela` ou automático | `seq_pedidos` |
| Enum | nome descritivo | `status_pedido` |

### Modelagem

- Normalize para evitar redundância; desnormalize quando performance justificar.
- Use `TIMESTAMPTZ` em vez de `TIMESTAMP` para evitar bugs de timezone.
- Prefira `BIGINT` para primary keys em tabelas que podem crescer muito.
- Prefira `IDENTITY` (`GENERATED ALWAYS AS IDENTITY`) sobre `SERIAL`.
- Defina `DEFAULT` e `NOT NULL` onde apropriado.
- Documente decisões de modelagem em comentários:

```sql
COMMENT ON TABLE clientes IS 'Cadastro de clientes ativos e inativos';
COMMENT ON COLUMN clientes.saldo IS 'Saldo em BRL, atualizado a cada transação';
```

---

**Referências:**

- [Documentação Oficial do PostgreSQL](https://www.postgresql.org/docs/current/)
- [SQL Standard (Wikipedia)](https://en.wikipedia.org/wiki/SQL:2016)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [Modern SQL](https://modern-sql.com/)
- [explain.depesz.com](https://explain.depesz.com/)
