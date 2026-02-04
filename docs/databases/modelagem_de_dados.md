# Modelagem de Dados — princípios e padrões

A modelagem de dados organiza como informação é armazenada e consultada. Bons modelos melhoram desempenho, manutenção e evolutividade.

## Conceitos-chave

- **Entidades e atributos**: tabelas e colunas.
- **Relacionamentos**: 1:1, 1:N, N:N (tabelas de junção).
- **Normalização**: reduzir redundância (1NF, 2NF, 3NF) até o ponto onde consultas e desempenho ainda fazem sentido.
- **Denormalização**: intencional para velocidade em leituras, manter com processos de sincronização.

## Modelos comuns

- **OLTP (transacional)**: normalizado, integridade forte, índices em chaves e colunas de filtro.
- **OLAP / Data Warehouse**: esquemas em estrela ou floco de neve (star/snowflake), focados em agregações e leitura.

## Chaves e integridade

- Use `PRIMARY KEY` e `FOREIGN KEY` para garantir integridade referencial.
- Considere `UNIQUE` e `CHECK` para regras de domínio.

## Particionamento e sharding

- **Particionamento**: dividir tabelas grandes por range/list/hash para melhorar manutenção e consultas por faixa (ex.: particionar por data).
- **Sharding**: dividir dados entre nós; aumenta complexidade de transações e joins.

## Exemplos rápidos

1) Normalizado (clientes e pedidos):

```sql
CREATE TABLE clientes (id SERIAL PRIMARY KEY, nome TEXT NOT NULL);
CREATE TABLE pedidos (id SERIAL PRIMARY KEY, cliente_id INT REFERENCES clientes(id), total NUMERIC);
```

2) Esquema estrela (dimensão/ fatos):

```sql
CREATE TABLE dim_cliente (...);
CREATE TABLE dim_tempo (...);
CREATE TABLE fato_vendas (cliente_id INT, tempo_id INT, valor NUMERIC);
```

## Boas práticas

- Modele para casos de uso mais frequentes (leitura vs escrita).
- Documente o modelo e as razões por trás de escolhas de normalização/denormalização.
- Reavalie o modelo quando requisitos de volume ou consultas mudarem.

Referências: livros de modelagem relacional, artigos sobre OLTP vs OLAP.
