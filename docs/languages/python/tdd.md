# TDD — Test-Driven Development

TDD é um ciclo de desenvolvimento baseado em escrever testes antes do código: Red → Green → Refactor.

## Ciclo

1. Escreva um teste que falhe (Red).
2. Implemente o mínimo necessário para passar (Green).
3. Refatore mantendo testes verdes (Refactor).

## Exemplo rápido

```python
def test_soma():
    assert soma(2, 3) == 5

def soma(a, b):
    return a + b
```

## Dicas

- Comece por casos simples e aumente a complexidade.
- Evite lógica desnecessária no primeiro passo; foque em comportamento.
- Use testes como especificação executável.

Ferramentas: `pytest`, mocks com `unittest.mock`.
