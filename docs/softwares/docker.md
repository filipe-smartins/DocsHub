# Docker - Comandos Essenciais

Um guia rápido com os comandos mais utilizados no Docker.

## Índice

- [Containers](#containers)
- [Imagens](#imagens)
- [Docker Compose](#docker-compose)

## <a id="containers">Containers</a>

- **Listar containers ativos:**
  ```bash
  docker ps
  ```
- **Listar todos os containers (incluindo parados):**
  ```bash
  docker ps -a
  ```
- **Criar e iniciar um container:**
  ```bash
  docker run --name nome-container -d imagem
  ```
- **Parar um container:**
  ```bash
  docker stop id-ou-nome
  ```
- **Iniciar um container parado:**
  ```bash
  docker start id-ou-nome
  ```
- **Remover um container:**
  ```bash
  docker rm id-ou-nome
  ```
- **Executar comando dentro de um container:**
  ```bash
  docker exec -it nome-container bash
  ```

## <a id="imagens">Imagens</a>

- **Listar imagens locais:**
  ```bash
  docker images
  ```
- **Baixar uma imagem do Docker Hub:**
  ```bash
  docker pull nome-imagem
  ```
- **Remover uma imagem:**
  ```bash
  docker rmi nome-imagem
  ```
- **Construir uma imagem a partir de um Dockerfile:**
  ```bash
  docker build -t nome-tag .
  ```

## <a id="docker-compose">Docker Compose</a>

- **Subir serviços definidos no docker-compose.yml:**
  ```bash
  docker-compose up -d
  ```
- **Parar e remover serviços:**
  ```bash
  docker-compose down
  ```
- **Ver logs dos serviços:**
  ```bash
  docker-compose logs -f
  ```


# Docker Avançado — imagens, segurança e otimização

Este guia cobre temas avançados para construir, otimizar e operar containers Docker em produção.

## Multi-stage builds

- Use multi-stage builds para reduzir o tamanho da imagem e separar etapas de build e runtime.

```dockerfile
FROM python:3.11-slim as build
WORKDIR /app
COPY pyproject.toml setup.cfg ./
RUN pip install --upgrade pip && pip install build
COPY . .
RUN pip wheel --no-deps --wheel-dir /wheels .

FROM python:3.11-slim
COPY --from=build /wheels /wheels
RUN pip install --no-deps /wheels/*.whl
COPY . /app
CMD ["myapp"]
```

## Camadas e cache

- Ordene instruções para maximizar cache: arquivos raramente mudados (instalação de dependências) primeiro; cópias de código por último.
- Use `--cache-from` para builds em CI e `buildx` com cache remoto.

## BuildKit e `docker buildx`

- Habilite BuildKit para builds paralelos, cache avançado e `--mount=type=cache`.

## Redução de tamanho de imagem

- Use imagens base reduzidas (`slim`, `alpine` quando compatível).
- Remova ferramentas de build na etapa final (multi-stage).
- Minimize dependências e arquivos copiados (`.dockerignore`).

## Segurança de imagens

- Execute processos como usuário não-root (`USER 1000`).
- Assine e verifique imagens (Docker Content Trust/Notary, cosign).
- Escaneie imagens por vulnerabilidades (Trivy, Clair, Snyk) no pipeline.

## Volumes, dados e backups

- Prefira volumes nomeados para dados persistentes.
- Para bancos, use backups regulares fora do container; não confie em volumes locais para RTO/RPO críticos.

## Redes e isolamento

- Use redes bridge customizadas para isolar conjuntos de serviços.
- Para comunicação entre stacks, considere redes overlay (Swarm) ou CNI com Kubernetes.

## Healthchecks e readiness

- Utilize `HEALTHCHECK` nas imagens para que orquestradores e pipelines detectem instâncias saudáveis.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s CMD curl -f http://localhost:8080/health || exit 1
```

## Limites de recursos

- Configure `--memory` e `--cpus` no runtime ou orquestrador; não dependa apenas do cgroup padrão.

## Runtime e compatibilidade

- Teste imagens em runtimes distintos (Docker Engine, containerd) e em ambientes com SELinux/AppArmor.

## CI/CD e imagens

- Faça builds reprodutíveis (tags imutáveis) e publique em registry privado (registry, GCR, ECR, ACR).
- Use `latest` apenas para desenvolvimento; em produção use tags semânticas ou digestos (`@sha256:`).

## Observabilidade e logs

- Redirecione logs para stdout/stderr; agregue com Fluentd/Fluent Bit/Logstash.
- Exporte métricas via endpoint (Prometheus) e monitore containers em nível de host.

## Boas práticas rápidas

- Tenha `.dockerignore` para excluir arquivos desnecessários.
- Evite segredos em imagens; injete via runtime (secrets, env vars, vault).
- Recrie imagens e teste em staging antes de promover para produção.

Referências: documentação Docker, BuildKit, e guias dos scanners (Trivy, Snyk).


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
