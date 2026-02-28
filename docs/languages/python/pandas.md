# Pandas — Guia completo do básico ao avançado

`pandas` é a biblioteca padrão para manipulação de dados tabulares em Python. Construída sobre NumPy, oferece duas estruturas principais — **Series** (1D) e **DataFrame** (2D) — e um ecossistema rico para leitura, limpeza, transformação, análise e exportação de dados.

---

## 1. Instalação

```bash
pip install pandas

# com dependências extras comuns
pip install pandas openpyxl sqlalchemy pyarrow fastparquet
```

> **Dica:** `openpyxl` é necessário para ler/escrever `.xlsx`, `pyarrow`/`fastparquet` para Parquet, e `sqlalchemy` para integração com bancos SQL.

---

## 2. Estruturas de dados fundamentais

### Series

Vetor unidimensional com rótulos (index):

```python
import pandas as pd
import numpy as np

s = pd.Series([10, 20, 30], index=['a', 'b', 'c'], name='valores')
s['b']       # 20
s.dtype      # int64
s.values     # array([10, 20, 30])
s.index      # Index(['a', 'b', 'c'], dtype='object')
```

### DataFrame

Tabela bidimensional (colunas tipadas + index de linhas):

```python
df = pd.DataFrame({
    'nome':  ['Ana', 'Bruno', 'Carla'],
    'idade': [28, 35, 42],
    'salario': [5500.0, 7200.0, 9100.0]
})

#     nome  idade  salario
# 0    Ana     28   5500.0
# 1  Bruno     35   7200.0
# 2  Carla     42   9100.0
```

### Criação a partir de diferentes fontes

```python
# a partir de lista de dicts
df = pd.DataFrame([{'a': 1, 'b': 2}, {'a': 3, 'b': 4}])

# a partir de array NumPy
df = pd.DataFrame(np.random.randn(5, 3), columns=['x', 'y', 'z'])

# a partir de dict de Series
df = pd.DataFrame({'col1': pd.Series([1, 2]), 'col2': pd.Series([3, 4])})
```

---

## 3. Leitura e escrita de dados

### CSV

```python
# leitura
df = pd.read_csv('dados.csv')
df = pd.read_csv('dados.csv', sep=';', encoding='latin-1', decimal=',')
df = pd.read_csv('dados.csv', usecols=['nome', 'valor'], nrows=1000)

# escrita
df.to_csv('saida.csv', index=False, encoding='utf-8')
```

### Excel

```python
# leitura (requer openpyxl)
df = pd.read_excel('planilha.xlsx', sheet_name='Vendas')
df = pd.read_excel('planilha.xlsx', sheet_name=0, skiprows=2, header=0)

# múltiplas abas
sheets = pd.read_excel('planilha.xlsx', sheet_name=None)  # dict de DataFrames

# escrita
with pd.ExcelWriter('relatorio.xlsx', engine='openpyxl') as writer:
    df_vendas.to_excel(writer, sheet_name='Vendas', index=False)
    df_custos.to_excel(writer, sheet_name='Custos', index=False)
```

### JSON

```python
df = pd.read_json('dados.json')
df = pd.read_json('dados.json', orient='records', lines=True)  # JSON Lines

df.to_json('saida.json', orient='records', indent=2, force_ascii=False)
```

### Parquet (recomendado para grandes volumes)

```python
# leitura (requer pyarrow ou fastparquet)
df = pd.read_parquet('dados.parquet')
df = pd.read_parquet('dados.parquet', columns=['col1', 'col2'])  # leitura seletiva

# escrita
df.to_parquet('saida.parquet', index=False, compression='snappy')
```

### SQL

```python
from sqlalchemy import create_engine

engine = create_engine('postgresql://user:pass@localhost:5432/mydb')

# leitura
df = pd.read_sql('SELECT * FROM vendas WHERE ano = 2025', engine)
df = pd.read_sql_table('clientes', engine)

# escrita
df.to_sql('resultados', engine, if_exists='replace', index=False, chunksize=5000)
# if_exists: 'fail' | 'replace' | 'append'
```

### HTML e Clipboard

```python
# ler tabelas de página web
tables = pd.read_html('https://exemplo.com/tabela.html')
df = tables[0]

# copiar/colar da área de transferência
df = pd.read_clipboard()
df.to_clipboard(index=False)
```

### Feather e HDF5

```python
# Feather — rápido para I/O entre Python e R
df.to_feather('dados.feather')
df = pd.read_feather('dados.feather')

# HDF5 — para datasets muito grandes
df.to_hdf('dados.h5', key='tabela', mode='w')
df = pd.read_hdf('dados.h5', key='tabela')
```

---

## 4. Exploração e inspeção de dados

```python
df.shape             # (linhas, colunas)
df.columns           # nomes das colunas
df.dtypes            # tipo de cada coluna
df.index             # índice (RangeIndex, DatetimeIndex, etc.)
len(df)              # número de linhas

df.head(10)          # primeiras 10 linhas
df.tail(5)           # últimas 5 linhas
df.sample(3)         # 3 linhas aleatórias
df.sample(frac=0.1)  # 10% das linhas

df.info()            # resumo: tipos, não-nulos, memória
df.describe()        # estatísticas numéricas (count, mean, std, min, quartis, max)
df.describe(include='all')  # inclui colunas categóricas/object

df.nunique()         # valores únicos por coluna
df['col'].value_counts()              # contagem de frequência
df['col'].value_counts(normalize=True) # frequência relativa (%)
```

### Estatísticas rápidas

```python
df['salario'].mean()      # média
df['salario'].median()    # mediana
df['salario'].std()       # desvio padrão
df['salario'].var()       # variância
df['salario'].min()       # mínimo
df['salario'].max()       # máximo
df['salario'].sum()       # soma
df['salario'].quantile([0.25, 0.5, 0.75])  # quartis

df.corr(numeric_only=True)   # matriz de correlação
df.cov(numeric_only=True)    # matriz de covariância
```

---

## 5. Seleção e indexação

### Seleção de colunas

```python
df['nome']                # Series
df[['nome', 'idade']]     # DataFrame com 2 colunas
df.nome                   # acesso por atributo (evitar — conflita com métodos)
```

### Seleção por rótulo — `loc`

```python
df.loc[0]                        # linha com index 0
df.loc[0:4]                      # linhas 0 a 4 (inclusive!)
df.loc[0, 'nome']                # célula específica
df.loc[:, 'nome':'salario']      # todas as linhas, colunas de 'nome' até 'salario'
df.loc[df['idade'] > 30]         # filtragem booleana
df.loc[df['idade'] > 30, 'nome'] # filtragem + seleção de coluna
```

### Seleção por posição — `iloc`

```python
df.iloc[0]              # primeira linha
df.iloc[0:5]            # linhas 0 a 4 (exclusivo no fim, como Python padrão)
df.iloc[0, 1]           # linha 0, coluna 1
df.iloc[:, 0:2]         # todas as linhas, colunas 0 e 1
df.iloc[-1]             # última linha
df.iloc[::2]            # linhas pares (step=2)
```

### Seleção condicional avançada

```python
# múltiplas condições (use & para AND, | para OR, ~ para NOT)
df.loc[(df['idade'] > 30) & (df['salario'] >= 7000)]
df.loc[(df['cidade'] == 'SP') | (df['cidade'] == 'RJ')]
df.loc[~df['nome'].str.contains('Silva')]

# isin — equivalente a SQL IN
df.loc[df['estado'].isin(['SP', 'RJ', 'MG'])]

# between
df.loc[df['idade'].between(25, 40)]

# query — sintaxe expressiva (strings)
df.query('idade > 30 and salario >= 7000')
df.query('cidade in ["SP", "RJ"]')
df.query('nome.str.startswith("A")', engine='python')
```

### Setando valores

```python
df.loc[df['idade'] > 60, 'categoria'] = 'sênior'
df.iloc[0, 2] = 9999
df.at[0, 'nome'] = 'Ana Maria'     # acesso escalar rápido (por rótulo)
df.iat[0, 0] = 'Ana Maria'         # acesso escalar rápido (por posição)
```

---

## 6. Filtragem e ordenação

### Filtragem de linhas

```python
# booleana
df[df['salario'] > 5000]

# com método query
df.query('salario > 5000 and idade < 40')

# nlargest / nsmallest — top N
df.nlargest(10, 'salario')
df.nsmallest(5, 'idade')
```

### Ordenação

```python
df.sort_values('salario', ascending=False)
df.sort_values(['estado', 'salario'], ascending=[True, False])

df.sort_index()              # ordenar pelo índice
df.sort_index(ascending=False)
```

### Remoção de duplicatas

```python
df.drop_duplicates()                           # todas as colunas
df.drop_duplicates(subset=['email'])           # por coluna específica
df.drop_duplicates(subset=['email'], keep='last')  # manter última ocorrência
df.duplicated(subset=['email']).sum()          # contar duplicatas
```

---

## 7. Manipulação de colunas

### Criar e modificar colunas

```python
# coluna simples
df['total'] = df['qtd'] * df['preco']

# com condição — np.where
df['faixa'] = np.where(df['idade'] >= 30, 'adulto', 'jovem')

# com múltiplas condições — np.select
conditions = [
    df['idade'] < 18,
    df['idade'].between(18, 59),
    df['idade'] >= 60
]
choices = ['menor', 'adulto', 'idoso']
df['faixa_etaria'] = np.select(conditions, choices, default='desconhecido')

# com apply (flexível, porém mais lento)
df['nome_upper'] = df['nome'].apply(str.upper)
df['desconto'] = df.apply(lambda row: row['total'] * 0.1 if row['categoria'] == 'VIP' else 0, axis=1)

# assign — retorna novo DataFrame (bom para encadear)
df = (df
    .assign(total=lambda x: x['qtd'] * x['preco'])
    .assign(imposto=lambda x: x['total'] * 0.15)
)
```

### Renomear colunas

```python
df.rename(columns={'nome': 'nome_completo', 'idade': 'anos'}, inplace=True)

# renomear todas de uma vez
df.columns = ['col_a', 'col_b', 'col_c']

# padronizar nomes
df.columns = df.columns.str.lower().str.replace(' ', '_').str.strip()
```

### Remover colunas e linhas

```python
df.drop(columns=['col_temporaria', 'col_debug'])
df.drop(columns=['col'], inplace=True)

# remover linhas por índice
df.drop(index=[0, 5, 10])
```

### Reordenar colunas

```python
df = df[['id', 'nome', 'email', 'salario']]

# mover coluna para o início
cols = ['nova_col'] + [c for c in df.columns if c != 'nova_col']
df = df[cols]
```

---

## 8. Tratamento de dados ausentes (NaN / None)

### Detecção

```python
df.isna()              # DataFrame booleano
df.isna().sum()        # contagem de NaN por coluna
df.isna().sum().sum()  # total geral de NaN
df.isna().mean()       # proporção de NaN por coluna (0-1)

df.notna()             # inverso de isna()
df['col'].isna().any() # True se houver pelo menos um NaN na coluna
```

### Remoção

```python
df.dropna()                              # remove linhas com qualquer NaN
df.dropna(subset=['email', 'telefone'])  # remove se NaN nessas colunas
df.dropna(how='all')                     # remove apenas se TODAS forem NaN
df.dropna(thresh=3)                      # mantém se tiver ao menos 3 valores não-NaN
```

### Preenchimento

```python
df['salario'].fillna(0)                          # valor fixo
df['salario'].fillna(df['salario'].mean())       # média
df['salario'].fillna(df['salario'].median())     # mediana
df['cidade'].fillna('Não informado')             # texto padrão

df.fillna(method='ffill')   # forward fill (valor anterior)
df.fillna(method='bfill')   # backward fill (valor seguinte)

# interpolação numérica
df['valor'].interpolate(method='linear')
df['valor'].interpolate(method='time')   # baseado em DatetimeIndex
```

### Substituição

```python
df.replace({'M': 'Masculino', 'F': 'Feminino'})
df['status'].replace({'A': 'Ativo', 'I': 'Inativo'}, inplace=True)
df.replace([np.inf, -np.inf], np.nan)    # infinitos → NaN
df.replace(r'^\s*$', np.nan, regex=True) # strings vazias → NaN
```

---

## 9. Tipos de dados (dtypes) e conversão

### Verificar tipos

```python
df.dtypes
df['col'].dtype
df.select_dtypes(include=['number'])          # só colunas numéricas
df.select_dtypes(include=['object'])          # só colunas texto
df.select_dtypes(exclude=['datetime64'])      # excluir datetime
```

### Converter tipos

```python
df['idade'] = df['idade'].astype(int)
df['preco'] = df['preco'].astype(float)
df['ativo'] = df['ativo'].astype(bool)
df['codigo'] = df['codigo'].astype(str)

# conversão segura (erros → NaN)
df['valor'] = pd.to_numeric(df['valor'], errors='coerce')

# datas
df['data'] = pd.to_datetime(df['data'])
df['data'] = pd.to_datetime(df['data'], format='%d/%m/%Y')
df['data'] = pd.to_datetime(df['data'], errors='coerce')  # inválidas → NaT

# categoria (reduz memória significativamente)
df['estado'] = df['estado'].astype('category')

# downcast numérico para economizar memória
df['qtd'] = pd.to_numeric(df['qtd'], downcast='integer')
df['valor'] = pd.to_numeric(df['valor'], downcast='float')
```

### Nullable dtypes (pandas >= 1.0)

```python
# dtypes nullable — suportam pd.NA em vez de np.nan
df['idade'] = df['idade'].astype(pd.Int64Dtype())   # ou 'Int64'
df['nome'] = df['nome'].astype(pd.StringDtype())     # ou 'string'
df['ativo'] = df['ativo'].astype(pd.BooleanDtype())  # ou 'boolean'
```

---

## 10. Manipulação de strings

Acessível via `.str` em colunas do tipo `object` ou `string`:

```python
df['nome'].str.lower()
df['nome'].str.upper()
df['nome'].str.title()
df['nome'].str.strip()             # remover espaços nas bordas
df['nome'].str.lstrip()            # espaços à esquerda
df['nome'].str.rstrip()            # espaços à direita

df['nome'].str.contains('Silva')               # booleano
df['nome'].str.contains('silva', case=False)   # case insensitive
df['nome'].str.startswith('A')
df['nome'].str.endswith('Jr')

df['nome'].str.replace('Sr.', '', regex=False)
df['nome'].str.replace(r'\d+', '', regex=True)  # remover números

df['nome'].str.len()              # comprimento
df['nome'].str.split(' ')        # divide em lista
df['nome'].str.split(' ').str[0] # primeiro nome
df['nome'].str.cat(df['sobrenome'], sep=' ')  # concatenar colunas

df['email'].str.extract(r'@(\w+\.\w+)')  # extrair com regex (grupo de captura)
df['texto'].str.findall(r'\b\w{5,}\b')   # encontrar todas as palavras com 5+ chars

df['cpf'].str.zfill(11)           # preencher zeros à esquerda
df['codigo'].str.pad(10, side='left', fillchar='0')
df['nome'].str.slice(0, 5)        # substring
```

---

## 11. Manipulação de datas e tempo

### Conversão e criação

```python
df['data'] = pd.to_datetime(df['data'])
df['data'] = pd.to_datetime(df['data'], format='%d/%m/%Y %H:%M:%S')

# criar range de datas
datas = pd.date_range(start='2025-01-01', end='2025-12-31', freq='D')
datas = pd.date_range(start='2025-01-01', periods=12, freq='MS')  # início de cada mês
```

### Acessor `.dt`

```python
df['ano']       = df['data'].dt.year
df['mes']       = df['data'].dt.month
df['dia']       = df['data'].dt.day
df['hora']      = df['data'].dt.hour
df['minuto']    = df['data'].dt.minute
df['segundo']   = df['data'].dt.second
df['dia_semana'] = df['data'].dt.day_name()          # 'Monday', etc.
df['trimestre'] = df['data'].dt.quarter
df['semana_ano'] = df['data'].dt.isocalendar().week   # ISO week
df['is_fim_semana'] = df['data'].dt.dayofweek >= 5    # sábado=5, domingo=6

df['data_str'] = df['data'].dt.strftime('%d/%m/%Y')
```

### Aritmética com datas

```python
df['dias_desde_compra'] = (pd.Timestamp.now() - df['data_compra']).dt.days
df['data_vencimento'] = df['data_compra'] + pd.Timedelta(days=30)
df['data_vencimento'] = df['data_compra'] + pd.DateOffset(months=1)

# diferença entre duas colunas
df['duracao'] = df['fim'] - df['inicio']            # Timedelta
df['duracao_horas'] = df['duracao'].dt.total_seconds() / 3600
```

### Filtragem por data

```python
df[df['data'] >= '2025-01-01']
df[df['data'].between('2025-01-01', '2025-06-30')]
df[df['data'].dt.year == 2025]
df[df['data'].dt.month.isin([1, 2, 3])]  # Q1

# com DatetimeIndex
df = df.set_index('data')
df['2025']                  # todo o ano
df['2025-03']               # todo o mês de março
df['2025-01':'2025-06']     # primeiro semestre
```

---

## 12. Agrupamento e agregação (`groupby`)

### Básico

```python
df.groupby('categoria')['valor'].sum()
df.groupby('categoria')['valor'].mean()
df.groupby('categoria').size()                 # contagem de linhas por grupo
df.groupby('categoria')['valor'].count()       # contagem de não-NaN

df.groupby(['estado', 'cidade'])['valor'].sum()
```

### Múltiplas agregações

```python
# agg com dict
df.groupby('categoria').agg({
    'valor': ['sum', 'mean', 'count'],
    'qtd':   'sum'
})

# named aggregation (pandas >= 0.25) — resultado com colunas planas
df.groupby('categoria').agg(
    total_valor=('valor', 'sum'),
    media_valor=('valor', 'mean'),
    qtd_vendas=('valor', 'count'),
    total_qtd=('qtd', 'sum')
).reset_index()
```

### Funções customizadas

```python
df.groupby('categoria')['valor'].agg(lambda x: x.max() - x.min())  # amplitude

def coef_variacao(x):
    return x.std() / x.mean() if x.mean() != 0 else 0

df.groupby('categoria')['valor'].agg(coef_variacao)
```

### Transform — aplicar e manter dimensão original

```python
# média do grupo em cada linha (útil para normalização)
df['media_grupo'] = df.groupby('categoria')['valor'].transform('mean')

# percentual em relação ao grupo
df['pct_grupo'] = df['valor'] / df.groupby('categoria')['valor'].transform('sum')

# z-score dentro do grupo
df['zscore'] = df.groupby('categoria')['valor'].transform(
    lambda x: (x - x.mean()) / x.std()
)
```

### Filter — filtrar grupos inteiros

```python
# manter apenas grupos com mais de 10 registros
df.groupby('categoria').filter(lambda x: len(x) > 10)

# manter grupos cuja média é maior que 1000
df.groupby('categoria').filter(lambda x: x['valor'].mean() > 1000)
```

---

## 13. Merge, Join e Concatenação

### Merge (SQL-like joins)

```python
# inner join (padrão)
resultado = pd.merge(vendas, clientes, on='cliente_id')

# left join
resultado = pd.merge(vendas, clientes, on='cliente_id', how='left')

# right, outer
resultado = pd.merge(vendas, clientes, on='cliente_id', how='outer')

# chaves com nomes diferentes
resultado = pd.merge(vendas, clientes, left_on='cod_cli', right_on='id_cliente')

# merge com múltiplas chaves
resultado = pd.merge(df1, df2, on=['ano', 'mes'])

# indicador de origem (similar a SQL COALESCE debug)
resultado = pd.merge(df1, df2, on='id', how='outer', indicator=True)
# _merge: 'left_only', 'right_only', 'both'

# sufixos para colunas homônimas
resultado = pd.merge(df1, df2, on='id', suffixes=('_venda', '_compra'))

# validação de cardinalidade
resultado = pd.merge(df1, df2, on='id', validate='one_to_many')
# opções: 'one_to_one', 'one_to_many', 'many_to_one', 'many_to_many'
```

### Join (por índice)

```python
df1.join(df2, how='left')                    # join pelo índice
df1.join(df2, on='key', how='inner')         # df1['key'] → df2.index
```

### Concatenação

```python
# empilhar verticalmente (linhas)
df_total = pd.concat([df1, df2, df3], ignore_index=True)

# lado a lado (colunas)
df_wide = pd.concat([df_a, df_b], axis=1)

# com chave identificadora
df_all = pd.concat([df1, df2], keys=['fonte1', 'fonte2'])
```

---

## 14. Pivot, Melt e Reshaping

### Pivot Table (tabela dinâmica)

```python
tabela = df.pivot_table(
    values='valor',
    index='categoria',
    columns='mes',
    aggfunc='sum',
    fill_value=0,
    margins=True,          # totais nas bordas
    margins_name='Total'
)
```

### Pivot simples (sem agregação)

```python
# cada combinação index/columns deve ser única
df.pivot(index='data', columns='produto', values='vendas')
```

### Melt — wide → long (derreter)

```python
df_long = df.melt(
    id_vars=['nome', 'ano'],       # colunas que ficam fixas
    value_vars=['jan', 'fev', 'mar'],  # colunas que viram linhas
    var_name='mes',
    value_name='valor'
)
```

### Stack / Unstack

```python
# stack: colunas → níveis do index (wide → long)
df.stack()

# unstack: nível do index → colunas (long → wide)
df.unstack()
df.unstack(level=0, fill_value=0)
```

### Crosstab

```python
pd.crosstab(df['genero'], df['escolaridade'])
pd.crosstab(df['genero'], df['escolaridade'], normalize='index')  # % por linha
pd.crosstab(df['genero'], df['escolaridade'], values=df['salario'], aggfunc='mean')
```

### Explode — expandir listas em linhas

```python
# coluna com listas → uma linha por elemento
df = pd.DataFrame({'nome': ['Ana', 'Bruno'], 'tags': [['py', 'sql'], ['js']]})
df.explode('tags')
#     nome tags
# 0    Ana   py
# 0    Ana  sql
# 1  Bruno   js
```

---

## 15. Window Functions (Janelas)

### Rolling (janela deslizante)

```python
df['media_movel_7d'] = df['vendas'].rolling(window=7).mean()
df['soma_movel_30d'] = df['vendas'].rolling(window=30).sum()
df['std_movel'] = df['vendas'].rolling(window=7).std()
df['min_movel'] = df['vendas'].rolling(window=7).min()

# janela centrada
df['media_centrada'] = df['vendas'].rolling(window=7, center=True).mean()

# min_periods — permite calcular mesmo com menos dados no início
df['media_movel'] = df['vendas'].rolling(window=7, min_periods=1).mean()
```

### Expanding (acumulado)

```python
df['soma_acumulada'] = df['vendas'].expanding().sum()
df['media_acumulada'] = df['vendas'].expanding().mean()
df['max_historico'] = df['vendas'].expanding().max()
```

### EWM (Média Móvel Exponencial)

```python
df['ewm_vendas'] = df['vendas'].ewm(span=7).mean()
df['ewm_vendas'] = df['vendas'].ewm(alpha=0.3).mean()
```

### Shift e Diff (deslocamento e diferença)

```python
df['vendas_ontem'] = df['vendas'].shift(1)       # lag de 1 período
df['vendas_amanha'] = df['vendas'].shift(-1)     # lead de 1 período
df['variacao_diaria'] = df['vendas'].diff(1)     # diferença entre períodos
df['variacao_pct'] = df['vendas'].pct_change()   # variação percentual

# rank
df['rank_vendas'] = df['vendas'].rank(ascending=False, method='dense')

# cumsum, cumprod, cummax, cummin
df['acumulado'] = df['vendas'].cumsum()
df['produto_acum'] = df['retorno'].cumprod()
```

---

## 16. Resample — Reamostragem temporal

É necessário um `DatetimeIndex`:

```python
df = df.set_index('data')

# downsampling: diário → mensal
df.resample('ME').sum()          # soma mensal ('ME' = month end)
df.resample('ME').agg({'vendas': 'sum', 'preco': 'mean'})

# downsampling: diário → semanal
df.resample('W').mean()

# upsampling: mensal → diário (com interpolação)
df.resample('D').interpolate(method='linear')

# frequências comuns:
# 'D' = dia, 'W' = semana, 'ME' = fim do mês, 'MS' = início do mês
# 'QE' = fim do trimestre, 'YE' = fim do ano
# 'h' = hora, 'min' = minuto, 'B' = dia útil
```

---

## 17. Multi-Index (índices hierárquicos)

### Criação

```python
arrays = [['SP', 'SP', 'RJ', 'RJ'], ['2024', '2025', '2024', '2025']]
index = pd.MultiIndex.from_arrays(arrays, names=['estado', 'ano'])
df = pd.DataFrame({'vendas': [100, 150, 80, 120]}, index=index)

# via groupby
df_grouped = df.groupby(['estado', 'cidade']).sum()
```

### Acesso

```python
df.loc['SP']               # todas as linhas de SP
df.loc[('SP', '2025')]     # linha específica
df.loc['SP':'RJ']          # slice de nível 0

# xs — cross-section
df.xs('2025', level='ano')              # todas as linhas de 2025
df.xs(('SP', '2025'), level=['estado', 'ano'])

# resetar para colunas normais
df.reset_index(inplace=True)
```

### Manipulação

```python
df.swaplevel()                     # trocar ordem dos níveis
df.sort_index(level='estado')      # ordenar por nível
df.index.get_level_values('estado') # valores de um nível
```

---

## 18. Categorical Data

Usado para colunas com número finito de valores distintos. Economiza memória e permite ordenação customizada.

```python
df['tamanho'] = pd.Categorical(
    df['tamanho'], 
    categories=['P', 'M', 'G', 'GG'], 
    ordered=True
)

df['tamanho'] = df['tamanho'].astype('category')

# acessor .cat
df['tamanho'].cat.categories         # categorias definidas
df['tamanho'].cat.codes              # códigos inteiros internos
df['tamanho'].cat.add_categories(['XG'])
df['tamanho'].cat.remove_unused_categories()
df['tamanho'].cat.rename_categories({'P': 'Pequeno', 'M': 'Médio'})
df['tamanho'].cat.reorder_categories(['P', 'M', 'G', 'GG'])

# com ordered=True, comparações funcionam
df[df['tamanho'] > 'M']   # filtra G e GG
df.sort_values('tamanho')  # ordena pela ordem definida, não alfabética
```

---

## 19. Apply, Map e Applymap

```python
# apply — aplica função a cada coluna (axis=0) ou linha (axis=1)
df['total'] = df.apply(lambda row: row['qtd'] * row['preco'], axis=1)
df[['a', 'b']].apply(np.sum, axis=0)  # soma de cada coluna

# map — aplica a cada elemento de uma Series (+ suporta dict)
df['status'] = df['cod_status'].map({1: 'Ativo', 2: 'Inativo', 3: 'Suspenso'})
df['nome'] = df['nome'].map(str.upper)

# applymap (DataFrame inteiro — renomeado para map em pandas >= 2.1)
df[['a', 'b']].applymap(lambda x: f'{x:.2f}')  # formatar tudo

# pipe — para encadear funções no DataFrame
def limpar(df):
    return df.dropna().drop_duplicates()

def enriquecer(df, taxa):
    df['imposto'] = df['valor'] * taxa
    return df

resultado = (df
    .pipe(limpar)
    .pipe(enriquecer, taxa=0.15)
)
```

---

## 20. Estilização e formatação de saída

### Display em notebooks

```python
pd.set_option('display.max_columns', None)       # mostrar todas as colunas
pd.set_option('display.max_rows', 100)
pd.set_option('display.max_colwidth', 80)
pd.set_option('display.float_format', '{:.2f}'.format)
pd.set_option('display.width', 200)

pd.reset_option('all')  # voltar ao padrão
```

### Styler (Jupyter)

```python
(df.style
    .format({'salario': 'R$ {:,.2f}', 'pct': '{:.1%}'})
    .highlight_max(subset=['salario'], color='lightgreen')
    .highlight_min(subset=['salario'], color='salmon')
    .bar(subset=['vendas'], color='#5fba7d')
    .background_gradient(subset=['score'], cmap='Blues')
    .set_caption('Relatório de Vendas')
)

# exportar para HTML
html = df.style.format({'valor': '{:.2f}'}).to_html()
```

---

## 21. Visualização integrada (plot)

`pandas` usa Matplotlib como backend padrão:

```python
import matplotlib.pyplot as plt

df['vendas'].plot(kind='line', title='Vendas ao longo do tempo', figsize=(10, 5))
plt.show()

df['categoria'].value_counts().plot(kind='bar')
df.plot(kind='scatter', x='idade', y='salario', alpha=0.5)
df[['receita', 'custo']].plot(kind='box')
df['valor'].plot(kind='hist', bins=30, edgecolor='black')
df.plot(kind='area', stacked=True)

# múltiplos subplots
df[['vendas', 'lucro', 'custo']].plot(subplots=True, layout=(1, 3), figsize=(15, 4))
```

### Usar Plotly como backend

```python
pd.options.plotting.backend = 'plotly'
df.plot(kind='scatter', x='idade', y='salario')  # gráfico interativo
```

---

## 22. Performance e otimização

### Regra geral: evite loops, prefira operações vetorizadas

```python
# ❌ LENTO — loop Python
for i, row in df.iterrows():
    df.at[i, 'total'] = row['qtd'] * row['preco']

# ✅ RÁPIDO — vetorizado
df['total'] = df['qtd'] * df['preco']
```

### Comparativo de velocidade

| Abordagem            | Velocidade relativa |
|----------------------|---------------------|
| Vetorizado (NumPy)   | ⚡⚡⚡ (mais rápido)  |
| `.values` + NumPy    | ⚡⚡⚡               |
| `apply()` com func C | ⚡⚡                 |
| `apply()` com lambda | ⚡                   |
| `itertuples()`       | 🐢                  |
| `iterrows()`         | 🐢🐢 (mais lento)   |

### Redução de memória

```python
def reduzir_memoria(df):
    """Reduz uso de memória fazendo downcast de tipos numéricos."""
    for col in df.select_dtypes(include=['int']).columns:
        df[col] = pd.to_numeric(df[col], downcast='integer')
    for col in df.select_dtypes(include=['float']).columns:
        df[col] = pd.to_numeric(df[col], downcast='float')
    for col in df.select_dtypes(include=['object']).columns:
        if df[col].nunique() / len(df) < 0.5:  # <50% valores únicos
            df[col] = df[col].astype('category')
    return df

print(f"Antes: {df.memory_usage(deep=True).sum() / 1e6:.1f} MB")
df = reduzir_memoria(df)
print(f"Depois: {df.memory_usage(deep=True).sum() / 1e6:.1f} MB")
```

### Leitura em chunks (arquivos muito grandes)

```python
chunks = []
for chunk in pd.read_csv('enorme.csv', chunksize=100_000):
    chunk = chunk[chunk['status'] == 'ativo']  # filtra durante leitura
    chunks.append(chunk)
df = pd.concat(chunks, ignore_index=True)
```

### Usar Parquet em vez de CSV

```python
# CSV: lento, pesado, sem tipos
# Parquet: rápido, compacto, preserva dtypes
df.to_parquet('dados.parquet')
df = pd.read_parquet('dados.parquet')
```

### eval e query para expressões grandes

```python
# eval — avalia expressão de string (evita cópias intermediárias)
df.eval('lucro = receita - custo', inplace=True)
df.eval('margem = lucro / receita', inplace=True)

# query — filtragem eficiente
df.query('receita > 10000 and estado == "SP"')
```

### Usar `NumPy` diretamente quando possível

```python
# np.where é mais rápido que apply para condicionais simples
df['flag'] = np.where(df['valor'] > 1000, 'alto', 'baixo')

# np.select para múltiplas condições (muito mais rápido que apply)
df['faixa'] = np.select(
    [df['valor'] < 100, df['valor'] < 1000, df['valor'] >= 1000],
    ['baixo', 'médio', 'alto'],
    default='desconhecido'
)
```

---

## 23. Pandas com grandes volumes — alternativas e integrações

### PyArrow backend (pandas >= 2.0)

```python
# usar Arrow como backend para tipos — menor memória, mais rápido
df = pd.read_csv('dados.csv', dtype_backend='pyarrow')
df = pd.read_parquet('dados.parquet', dtype_backend='pyarrow')

df.dtypes
# nome       string[pyarrow]
# idade      int64[pyarrow]
# salario    double[pyarrow]
```

### Dask — pandas distribuído

```python
import dask.dataframe as dd

ddf = dd.read_csv('enorme_*.csv')   # leitura lazy de múltiplos arquivos
resultado = ddf.groupby('categoria')['valor'].sum().compute()  # .compute() executa
```

### Polars — alternativa de alta performance

```python
import polars as pl

df = pl.read_csv('dados.csv')
resultado = (
    df
    .filter(pl.col('valor') > 100)
    .group_by('categoria')
    .agg(pl.col('valor').sum())
    .sort('valor', descending=True)
)
```

---

## 24. Padrões e receitas comuns

### Ler múltiplos arquivos e concatenar

```python
import glob

arquivos = glob.glob('dados/vendas_*.csv')
df = pd.concat([pd.read_csv(f) for f in arquivos], ignore_index=True)
```

### One-hot encoding

```python
df_encoded = pd.get_dummies(df, columns=['cidade', 'genero'], drop_first=True)
```

### Binning (faixas/categorizar valores contínuos)

```python
# faixas iguais
df['faixa_idade'] = pd.cut(df['idade'], bins=[0, 18, 30, 50, 100],
                           labels=['menor', 'jovem', 'adulto', 'sênior'])

# faixas por quantis
df['quartil_salario'] = pd.qcut(df['salario'], q=4, labels=['Q1', 'Q2', 'Q3', 'Q4'])
```

### Anti-join (linhas que NÃO estão no outro DataFrame)

```python
# IDs presentes em df1 mas ausentes em df2
df_anti = df1[~df1['id'].isin(df2['id'])]

# ou via merge + indicator
merged = pd.merge(df1, df2, on='id', how='left', indicator=True)
df_anti = merged[merged['_merge'] == 'left_only'].drop(columns='_merge')
```

### Flatten MultiIndex columns após groupby

```python
df_agg = df.groupby('cat').agg({'valor': ['sum', 'mean'], 'qtd': 'sum'})
df_agg.columns = ['_'.join(col).strip('_') for col in df_agg.columns]
df_agg.reset_index(inplace=True)
```

### Progress bar com tqdm

```python
from tqdm import tqdm
tqdm.pandas()

df['resultado'] = df['texto'].progress_apply(funcao_demorada)
```

### Comparar dois DataFrames

```python
# verificar se são iguais
df1.equals(df2)

# ver diferenças linha a linha
diff = df1.compare(df2)
```

---

## 25. Boas práticas

1. **Use `copy()` ao derivar DataFrames** — evitar `SettingWithCopyWarning`:

    ```python
    df_filtrado = df[df['status'] == 'ativo'].copy()
    df_filtrado['nova_col'] = 1  # seguro
    ```

2. **Prefira `assign()` para pipelines** — imutabilidade e encadeamento:

    ```python
    resultado = (df
        .query('valor > 0')
        .assign(total=lambda x: x['qtd'] * x['preco'])
        .groupby('categoria')
        .agg(total=('total', 'sum'))
        .reset_index()
        .sort_values('total', ascending=False)
    )
    ```

3. **Defina `dtypes` no momento da leitura**:

    ```python
    dtypes = {'id': 'int32', 'nome': 'string', 'valor': 'float32', 'status': 'category'}
    df = pd.read_csv('dados.csv', dtype=dtypes, parse_dates=['data_criacao'])
    ```

4. **Evite `inplace=True`** — dificulta encadeamento e debug.

5. **Use nomes descritivos para colunas** — padronize `snake_case`, sem acentos.

6. **Documente transformações complexas** — comente cada etapa do pipeline.

7. **Teste com `assert`** — validar premissas sobre os dados:

    ```python
    assert df['id'].is_unique, "IDs duplicados encontrados!"
    assert df['valor'].notna().all(), "Existem valores nulos em 'valor'"
    assert len(df) > 0, "DataFrame vazio!"
    assert set(df['status'].unique()) <= {'ativo', 'inativo'}, "Status inesperado"
    ```

8. **Use `pd.option_context` para configurações temporárias**:

    ```python
    with pd.option_context('display.max_rows', 200, 'display.float_format', '{:.4f}'.format):
        print(df.describe())
    ```

---

## 26. Cheat sheet — Referência rápida

| Operação | Código |
|---|---|
| Ler CSV | `pd.read_csv('f.csv')` |
| Ler Excel | `pd.read_excel('f.xlsx')` |
| Ler Parquet | `pd.read_parquet('f.parquet')` |
| Ler SQL | `pd.read_sql(query, engine)` |
| Info | `df.info()`, `df.describe()` |
| Filtrar | `df[df['col'] > x]`, `df.query(...)` |
| Selecionar | `df.loc[...]`, `df.iloc[...]` |
| Ordenar | `df.sort_values('col')` |
| Agrupar | `df.groupby('col').agg(...)` |
| Merge | `pd.merge(a, b, on='key')` |
| Concat | `pd.concat([a, b])` |
| Pivot | `df.pivot_table(...)` |
| Melt | `df.melt(...)` |
| Nulos | `df.isna()`, `df.fillna()`, `df.dropna()` |
| Duplicatas | `df.drop_duplicates()` |
| Tipos | `df.astype(...)`, `pd.to_numeric(...)` |
| Datas | `pd.to_datetime(...)`, `df['d'].dt.year` |
| Strings | `df['s'].str.lower()`, `.str.contains()` |
| Rolling | `df['c'].rolling(7).mean()` |
| Salvar CSV | `df.to_csv('f.csv', index=False)` |
| Salvar Parquet | `df.to_parquet('f.parquet')` |

---

Referências:

- Documentação oficial: [https://pandas.pydata.org/docs/](https://pandas.pydata.org/docs/)
- User Guide: [https://pandas.pydata.org/docs/user_guide/index.html](https://pandas.pydata.org/docs/user_guide/index.html)
- API Reference: [https://pandas.pydata.org/docs/reference/index.html](https://pandas.pydata.org/docs/reference/index.html)
- Pandas Cheat Sheet (oficial): [https://pandas.pydata.org/Pandas_Cheat_Sheet.pdf](https://pandas.pydata.org/Pandas_Cheat_Sheet.pdf)
