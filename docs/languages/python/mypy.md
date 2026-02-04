# mypy — Tipagem estática em Python

`mypy` permite checar anotações de tipo em Python para encontrar inconsistências antes da execução.

## Instalação

```bash
pip install mypy
```

## Anotações básicas

```python
def saudacao(nome: str) -> str:
    return f"Olá, {nome}"
```

## Uso

```bash
mypy src
```

## Configuração

Use `mypy.ini` ou `pyproject.toml` para opções (`ignore_missing_imports`, `strict`).

## Gradual typing

- Adote tipagem aos poucos em código legado; `# type: ignore` onde necessário.
- Utilize `typing` para tipos complexos (`List`, `Optional`, `TypedDict`, `Protocol`).

## Boas práticas

- Tipar interfaces públicas e funções críticas.
- Combine `mypy` com testes unitários para garantir correção.

Referências: https://mypy.readthedocs.io/
