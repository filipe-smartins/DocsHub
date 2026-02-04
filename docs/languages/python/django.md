# Django — Guia rápido de início e deployment

Django é um framework web de alto nível para Python com foco em rapidez de desenvolvimento e design limpo.

## Instalação e projeto

```bash
pip install django
django-admin startproject myproject
cd myproject
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
```

## Estrutura básica

- `manage.py`: utilitário de linha de comando.
- `myproject/settings.py`: configurações do projeto.
- `myproject/urls.py`: roteamento global.
- Apps: crie com `python manage.py startapp appname`.

## Comandos úteis

- `python manage.py makemigrations` — gerar migrations.
- `python manage.py migrate` — aplicar migrations.
- `python manage.py createsuperuser` — criar admin.
- `python manage.py collectstatic` — coletar arquivos estáticos para produção.

## Settings importantes para produção

- `DEBUG = False`
- Configure `ALLOWED_HOSTS` com domínios permitidos.
- Use `SECRET_KEY` seguro (não comitar no repo).
- Configure `STATIC_ROOT` e `MEDIA_ROOT` para servir arquivos estáticos/media.

## Deployment (resumo)

- Use Gunicorn (WSGI) ou Uvicorn+Gunicorn for ASGI apps.
- Front the app with Nginx; sirva static files via Nginx.
- Configure systemd para gerenciar o processo Gunicorn.
- Use conexões seguras (HTTPS) e variáveis de ambiente para credenciais.

## Boas práticas

- Separe configurações por ambiente (ex.: `settings/base.py`, `settings/production.py`).
- Use `django-environ` ou `python-dotenv` para variáveis de ambiente.
- Habilite logging e monitore erros (Sentry, etc.).

Referências: documentação oficial do Django (https://docs.djangoproject.com/).
