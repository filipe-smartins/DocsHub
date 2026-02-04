# Docker Compose — orquestrar multi-containers localmente

`docker-compose` simplifica o desenvolvimento local e pequenos ambientes multi-container. A versão atual é distribuída como `docker compose` (plugin) ou `docker-compose` (v1).

## Estrutura básica (`docker-compose.yml`)

```yaml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/app

  db:
    image: postgres:15
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

## `depends_on` e healthchecks

- `depends_on` controla a ordem de start, mas não garante que o serviço esteja pronto; combine com `healthcheck` e scripts de retry.

```yaml
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

## Variáveis de ambiente e `env_file`

- Use `env_file` para separar credenciais locais e não comitar `.env` no repositório.

## Perfis e sobreposição de configurações

- Use `docker-compose.override.yml` para sobrescrever configurações em desenvolvimento.
- Compose v2 suporta `profiles:` para ativar grupos de serviços.

## Volumes e bind mounts

- Em desenvolvimento use bind mounts para refletir código local dentro do container (`.:/app`).
- Em produção prefira imagens imutáveis e volumes gerenciados.

## Rede e descoberta

- Compose cria uma rede por projeto; serviços se comunicam pelo nome do serviço.

## Escalonamento local

- `docker compose up --scale worker=3` para múltiplas réplicas locais (útil para desenvolvimento e teste).

## Integração com CI

- Use `docker compose -f docker-compose.ci.yml up --build --abort-on-container-exit` em jobs de integração.

## Boas práticas

- Versione arquivos de compose; documente modo de uso em README.
- Evite usar `latest` em imagens; prefira tags específicas.
- Limpe volumes e redes com `docker compose down --volumes --rmi local` quando necessário.

Referências: documentação oficial do Docker Compose.
