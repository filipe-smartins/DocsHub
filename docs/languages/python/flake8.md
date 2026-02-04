# flake8 — Checagem de estilo e problemas estáticos

`flake8` combina PyFlakes, pycodestyle e McCabe para encontrar erros de sintaxe, estilo e complexidade.

## Instalação

```bash
pip install flake8
```

## Uso básico

```bash
flake8 src tests
```

## Configuração comum (`.flake8`)

- `max-line-length` — compatível com `black` use 88.
- `extend-ignore` — ignore E203 e W503 ao usar `black`.

## Plugins úteis

- `flake8-bugbear` — detecta padrões de código problemáticos.
- `flake8-docstrings` — checa docstrings (pydocstyle).
- `flake8-comprehensions` — melhorias em comprehensions.

## Boas práticas

- Rode `flake8` no CI e localmente via `pre-commit`.
- Conserte avisos progressivamente; prefira regras estritas em novos projetos.

Referências: https://flake8.pycqa.org/
