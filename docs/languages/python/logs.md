# Logging em Python

O módulo padrão `logging` fornece uma forma flexível de emitir mensagens de log em aplicações Python.

## Conceitos principais

- **Logger**: objeto que produz mensagens. Geralmente obtido por `logging.getLogger(__name__)`.
- **Handler**: destino das mensagens (console, arquivo, syslog, etc.).
- **Formatter**: formata a mensagem (timestamp, nível, origem).
- **Níveis**: DEBUG, INFO, WARNING, ERROR, CRITICAL.

## Exemplo mínimo

```python
import logging

logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(name)s: %(message)s')
logger = logging.getLogger(__name__)

logger.info('Iniciando aplicação')
try:
	1 / 0
except Exception:
	logger.exception('Erro inesperado')
```

## Log em arquivo com rotação

```python
from logging.handlers import RotatingFileHandler
handler = RotatingFileHandler('app.log', maxBytes=10_000_000, backupCount=5)
handler.setFormatter(logging.Formatter('%(asctime)s %(levelname)s %(message)s'))

root = logging.getLogger()
root.setLevel(logging.INFO)
root.addHandler(handler)
```

## Boas práticas

- Não use `print()` para logs em produção; use `logging`.
- Configure loggers por módulo com `logging.getLogger('meu.modulo')`.
- Redirecione logs de bibliotecas configurando handlers apenas no entrypoint da aplicação.
- Use `logger.exception()` dentro de exceções para incluir stacktrace.

Referências: documentação oficial do `logging` (https://docs.python.org/3/library/logging.html).
