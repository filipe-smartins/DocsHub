# Gunicorn — Servidor WSGI para aplicações Python

`gunicorn` é um servidor WSGI leve e amplamente usado para executar aplicações Python (Flask, Django, FastAPI via ASGI adapters, etc.).

## Instalação

```bash
pip install gunicorn
```

## Comando básico

Executar um app Flask (ex.: `app:app`):

```bash
gunicorn --bind 0.0.0.0:8000 app:app
```

Executar uma app Django (entry: `myproject.wsgi:application`):

```bash
gunicorn --workers 3 --bind 0.0.0.0:8000 myproject.wsgi:application
```

## Recomendações de configuração

- Workers: para CPU-bound use `gthread` ou `sync` com `--workers` ~= (2 x $CPU) + 1. Para I/O-bound use `--worker-class gevent` ou `gthread`.
- Timeout: ajuste `--timeout` para evitar workers zumbis em requests longos.
- Preload: `--preload` carrega a aplicação antes de forkar workers (útil para reduzir uso de memória com copy-on-write), mas não é seguro se o app mantém conexões abertas no startup.

## Exemplo de unit file systemd

```ini
[Unit]
Description=Gunicorn daemon for myproject
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/myproject
ExecStart=/path/to/venv/bin/gunicorn --workers 3 --bind unix:/run/gunicorn.sock myproject.wsgi:application

[Install]
WantedBy=multi-user.target
```

## Logs e diagnóstico

- Use `--access-logfile` e `--error-logfile` para arquivos separados.
- Monitore uso de CPU/memória dos workers; aumente `workers` se CPU estiver ociosa e latência alta.

Referências: documentação oficial do Gunicorn (https://gunicorn.org/).
