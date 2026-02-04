# Testes de Integração

Testes de integração verificam comportamento entre componentes ou com serviços externos (DB, APIs, filas).

## Estratégias

- Isolar dependências com containers (`docker-compose`) ou usar testes com bancos em memória.
- Usar fixtures para provisionar recursos e garantir limpeza.
- Separar testes de integração em `tests/integration/`.

## Exemplos de setup

Usando `docker-compose` para levantar um DB de teste:

```bash
docker-compose -f tests/docker-compose.yml up -d
pytest -m integration
docker-compose -f tests/docker-compose.yml down
```

Ou usar Testcontainers (Python) para criar instâncias temporárias.

## Boas práticas

- Execute integração em CI em runners com recursos (p. ex. matrix jobs).
- Marque testes lentos para rodar separadamente (`@pytest.mark.slow`).
- Certifique-se de teardown consistente para evitar dados residuais.

Referências: artigos sobre Testcontainers e documentação do pytest fixtures.
