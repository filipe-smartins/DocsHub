# SQL: principais tipos de queries

Este guia resume os tipos mais comuns de consultas SQL usadas no dia a dia.

## Índice

- [1) SELECT (leitura)](#1-select-leitura)
- [2) INSERT (inserção)](#2-insert-insercao)
- [3) UPDATE (atualização)](#3-update-atualizacao)
- [4) DELETE (remoção)](#4-delete-remocao)
- [5) JOIN (combinação de tabelas)](#5-join-combinacao-de-tabelas)
- [6) GROUP BY e agregações](#6-group-by-e-agregacoes)
- [7) HAVING (filtro pós-agregação)](#7-having-filtro-pos-agregacao)
- [8) ORDER BY e LIMIT](#8-order-by-e-limit)
- [9) CREATE / ALTER / DROP (DDL)](#9-create-alter-drop-ddl)
- [10) INSERT ... SELECT (carga a partir de consulta)](#10-insert-select)
- [11) TRANSACTION (controle transacional)](#11-transaction)
- [12) VIEW (consulta reutilizável)](#12-view)
- [Boas práticas rápidas](#boas-praticas-rapidas)

## <a id="1-select-leitura">1) SELECT (leitura)</a>

Usado para consultar dados.

Exemplos:

- Buscar todas as colunas:

	SELECT * FROM clientes;

- Selecionar colunas específicas:

	SELECT id, nome, email FROM clientes;

- Filtrar com WHERE:

	SELECT * FROM pedidos WHERE status = 'pago';

- Ordenar resultados:

	SELECT * FROM produtos ORDER BY preco DESC;

## <a id="2-insert-insercao">2) INSERT (inserção)</a>

Usado para adicionar novos registros.

Exemplo:

INSERT INTO clientes (nome, email, ativo)
VALUES ('Ana Silva', 'ana@exemplo.com', true);

## <a id="3-update-atualizacao">3) UPDATE (atualização)</a>

Usado para modificar registros existentes.

Exemplo:

UPDATE clientes
SET ativo = false
WHERE id = 10;

## <a id="4-delete-remocao">4) DELETE (remoção)</a>

Usado para remover registros.

Exemplo:

DELETE FROM clientes
WHERE id = 10;

## <a id="5-join-combinacao-de-tabelas">5) JOIN (combinação de tabelas)</a>

Combina registros de duas ou mais tabelas.

- INNER JOIN (apenas correspondências):

	SELECT c.nome, p.total
	FROM clientes c
	INNER JOIN pedidos p ON p.cliente_id = c.id;

- LEFT JOIN (tudo da esquerda e correspondências da direita):

	SELECT c.nome, p.total
	FROM clientes c
	LEFT JOIN pedidos p ON p.cliente_id = c.id;

## <a id="6-group-by-e-agregacoes">6) GROUP BY e agregações</a>

Agrupa registros e aplica funções agregadas.

Exemplo:

SELECT status, COUNT(*) AS total
FROM pedidos
GROUP BY status;

Funções comuns: COUNT, SUM, AVG, MIN, MAX.

## <a id="7-having-filtro-pos-agregacao">7) HAVING (filtro pós-agregação)</a>

Filtra grupos após agregação.

Exemplo:

SELECT status, COUNT(*) AS total
FROM pedidos
GROUP BY status
HAVING COUNT(*) > 5;

## <a id="8-order-by-e-limit">8) ORDER BY e LIMIT</a>

Define ordenação e limite de resultados.

Exemplo:

SELECT * FROM produtos
ORDER BY preco DESC
LIMIT 10;

## <a id="9-create-alter-drop-ddl">9) CREATE / ALTER / DROP (DDL)</a>

Comandos de definição de dados (estrutura do banco).

- CREATE TABLE:

	CREATE TABLE clientes (
		id SERIAL PRIMARY KEY,
		nome TEXT NOT NULL,
		email TEXT UNIQUE
	);

- ALTER TABLE:

	ALTER TABLE clientes ADD COLUMN telefone TEXT;

- DROP TABLE:

	DROP TABLE clientes;

## <a id="10-insert-select">10) INSERT ... SELECT (carga a partir de consulta)</a>

Insere dados com base no resultado de outra consulta.

Exemplo:

INSERT INTO clientes_ativos (id, nome)
SELECT id, nome FROM clientes WHERE ativo = true;

## <a id="11-transaction">11) TRANSACTION (controle transacional)</a>

Garante consistência ao agrupar comandos.

Exemplo:

BEGIN;
UPDATE contas SET saldo = saldo - 100 WHERE id = 1;
UPDATE contas SET saldo = saldo + 100 WHERE id = 2;
COMMIT;

Use ROLLBACK para desfazer em caso de erro.

## <a id="12-view">12) VIEW (consulta reutilizável)</a>

Cria uma visão para facilitar consultas.

Exemplo:

CREATE VIEW vw_pedidos_pagos AS
SELECT * FROM pedidos WHERE status = 'pago';

## <a id="boas-praticas-rapidas">Boas práticas rápidas</a>

- Evite SELECT * em produção; selecione apenas as colunas necessárias.
- Use índices para colunas usadas em filtros e junções.
- Prefira transações ao atualizar múltiplas tabelas.
