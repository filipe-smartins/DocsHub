# Pandas — Introdução rápida

`pandas` é a biblioteca padrão para manipulação de dados tabulares em Python (DataFrame/Series).

## Instalação

```bash
pip install pandas
```

## Leitura e escrita

```python
import pandas as pd
df = pd.read_csv('dados.csv')
df.to_csv('out.csv', index=False)
```

## Operações básicas

- `df.head()` / `df.tail()` — visualizar linhas.
- `df.info()` / `df.describe()` — resumo e tipos.
- Seleção: `df['col']`, `df[['a','b']]`, `df.loc[row, col]`, `df.iloc[i, j]`.
- Filtragem: `df[df['idade'] > 30]`.
- Agrupamento: `df.groupby('grupo').agg({'valor':'sum'})`.
- Merge / join: `pd.merge(left, right, on='id', how='inner')`.

## Exemplos úteis

```python
# criar coluna
df['total'] = df['qtd'] * df['preco']

# agrupar e ordenar
res = df.groupby('categoria')['total'].sum().reset_index().sort_values('total', ascending=False)
```

## Performance e dicas

- Especifique dtypes ao ler (`dtype=`) para reduzir memória.
- Use `chunksize` para arquivos muito grandes (`for chunk in pd.read_csv(..., chunksize=100000):`).
- Considere `categorical` para colunas com poucas categorias.
- Para processamento em escala, veja Dask ou Polars.

Referências: documentação oficial `pandas` (https://pandas.pydata.org/).
