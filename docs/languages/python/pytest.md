# pytest — Testes unitários em Python

`pytest` é uma ferramenta popular para escrever e executar testes unitários e funcionais em Python.

## Instalação

```bash
pip install pytest
```

## Estrutura de testes

- Coloque testes em um diretório `tests/`.
- Nomeie arquivos `test_*.py` ou `*_test.py`.
- Nomeie funções de teste com `test_`.

## Executando testes

```bash
pytest -q            # execução simples
pytest -k "login"   # rodar testes que combinam expressão
pytest -m integration # rodar testes marcados como integração
```

## Fixtures

Fixtures provêm setup/teardown reutilizável:

```python
import pytest

@pytest.fixture
def cliente():
    from myapp import create_app
    app = create_app(testing=True)
    return app.test_client()

def test_home(cliente):
    res = cliente.get('/')
    assert res.status_code == 200
```

## Marcação e seleção

Use marcadores para separar unit/integration/slow:

```python
import pytest

@pytest.mark.integration
def test_api():
    ...
```

E rode com `pytest -m integration`.

## Boas práticas

- Mantenha testes rápidos e determinísticos.
- Prefira testes unitários pequenos; use integração apenas quando necessário.
- Use `--maxfail=1` em CI para falhar cedo.
- Gere relatórios JUnit com `pytest --junitxml=report.xml` para CI.

Referências: https://docs.pytest.org/
