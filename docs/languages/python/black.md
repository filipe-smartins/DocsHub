# black — Formatador de código automático

`black` aplica um estilo consistente automaticamente, reduzindo debates de estilo em PRs.

## Instalação

```bash
pip install black
```

## Uso

```bash
black .           # formata todo o repositório
black src tests   # formatar pastas específicas
```

## Integração

- Execute `black --check` no CI para garantir formato.
- Use `pre-commit` para formatar automaticamente antes de commits.

## Configuração

Defina `line-length = 88` em `pyproject.toml` para compatibilidade comum.

## Boas práticas

- Deixe o `black` cuidar do formato; foque em estilo de alto nível (naming, arquitetura).

Referências: https://black.readthedocs.io/
