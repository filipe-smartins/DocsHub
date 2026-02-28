# Docker — Guia Completo do Básico ao Avançado

Guia abrangente cobrindo conceitos fundamentais, comandos, Dockerfile, networking, volumes, segurança, otimização e orquestração com Docker Compose.

---

## Índice

- [Conceitos Fundamentais](#conceitos-fundamentais)
- [Instalação](#instalacao)
- [Ciclo de Vida dos Containers](#ciclo-de-vida-dos-containers)
- [Comandos Essenciais — Containers](#containers)
- [Comandos Essenciais — Imagens](#imagens)
- [Inspecionar e Depurar](#inspecionar-e-depurar)
- [Logs](#logs)
- [Dockerfile — Referência Completa](#dockerfile-referencia-completa)
- [.dockerignore](#dockerignore)
- [Multi-stage Builds](#multi-stage-builds)
- [Camadas e Cache](#camadas-e-cache)
- [BuildKit e docker buildx](#buildkit-e-docker-buildx)
- [Builds Multi-plataforma](#builds-multi-plataforma)
- [Redução de Tamanho de Imagem](#reducao-de-tamanho-de-imagem)
- [Volumes e Persistência de Dados](#volumes-e-persistencia-de-dados)
- [Bind Mounts e tmpfs](#bind-mounts-e-tmpfs)
- [Networking](#networking)
- [Mapeamento de Portas](#mapeamento-de-portas)
- [Variáveis de Ambiente](#variaveis-de-ambiente)
- [Limites de Recursos](#limites-de-recursos)
- [Healthchecks](#healthchecks)
- [Segurança](#seguranca)
- [Docker Registry](#docker-registry)
- [Limpeza e Manutenção](#limpeza-e-manutencao)
- [Observabilidade e Logs em Produção](#observabilidade-e-logs-em-producao)
- [CI/CD com Docker](#cicd-com-docker)
- [Docker em Produção — Boas Práticas](#docker-em-producao-boas-praticas)
- [Docker Contexts](#docker-contexts)
- [Docker Compose — Orquestração Local](#docker-compose-orquestracao-local)
- [Troubleshooting](#troubleshooting)
- [Alternativas ao Docker](#alternativas-ao-docker)

---

## <a id="conceitos-fundamentais">Conceitos Fundamentais</a>

### O que é Docker?

Docker é uma plataforma de containerização que empacota aplicações e suas dependências em unidades isoladas chamadas **containers**. Containers compartilham o kernel do sistema operacional host, tornando-os mais leves e rápidos que máquinas virtuais.

### Containers vs Máquinas Virtuais

| Característica       | Container                      | VM                              |
|----------------------|--------------------------------|---------------------------------|
| Isolamento           | Processos e namespaces         | Hardware virtualizado completo  |
| Tamanho              | Megabytes                      | Gigabytes                       |
| Startup              | Segundos                       | Minutos                         |
| Performance          | Quase nativa                   | Overhead de hypervisor          |
| Kernel               | Compartilhado com o host       | Próprio por VM                  |
| Densidade            | Centenas por host              | Dezenas por host                |

### Arquitetura Docker

- **Docker Daemon (`dockerd`):** processo servidor que gerencia imagens, containers, redes e volumes.
- **Docker Client (`docker`):** CLI que envia comandos ao daemon via API REST.
- **Docker Registry:** repositório de imagens (Docker Hub, registries privados).
- **Imagem:** template read-only com camadas (layers) para criar containers.
- **Container:** instância executável de uma imagem.
- **Dockerfile:** arquivo texto com instruções para construir uma imagem.

### Namespaces e cgroups

- **Namespaces:** fornecem isolamento de PID, rede, mount, UTS, IPC e user.
- **cgroups (Control Groups):** limitam e monitoram uso de CPU, memória, I/O e rede por container.

---

## <a id="instalacao">Instalação</a>

### Linux (Ubuntu/Debian)

```bash
# Remover versões antigas
sudo apt-get remove docker docker-engine docker.io containerd runc

# Instalar dependências
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release

# Adicionar chave GPG oficial
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Adicionar repositório
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Permitir uso sem sudo
sudo usermod -aG docker $USER
newgrp docker
```

### macOS e Windows

- Instale o **Docker Desktop** a partir de [docker.com](https://www.docker.com/products/docker-desktop).
- No Windows, habilite **WSL 2** (recomendado) ou Hyper-V.

### Verificar instalação

```bash
docker --version
docker compose version
docker run hello-world
```

### Configurar Docker Daemon

Edite `/etc/docker/daemon.json` para personalizar comportamento:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "default-address-pools": [
    {"base": "172.80.0.0/16", "size": 24}
  ],
  "dns": ["8.8.8.8", "8.8.4.4"],
  "live-restore": true
}
```

```bash
sudo systemctl restart docker
```

---

## <a id="ciclo-de-vida-dos-containers">Ciclo de Vida dos Containers</a>

```
Created → Running → Paused → Running → Stopped → Removed
   ↓                                        ↑
   └──────────── Restart ──────────────────→ ┘
```

| Estado      | Descrição                                      | Comando para transição             |
|-------------|------------------------------------------------|------------------------------------|
| Created     | Container criado mas não iniciado              | `docker create`                    |
| Running     | Container em execução                          | `docker start` / `docker run`      |
| Paused      | Processos congelados (SIGSTOP via cgroups)     | `docker pause` / `docker unpause`  |
| Stopped     | Container parado (processo principal encerrou) | `docker stop` / `docker kill`      |
| Removed     | Container removido do sistema                  | `docker rm`                        |

---

## <a id="containers">Comandos Essenciais — Containers</a>

### Listar containers

```bash
# Containers em execução
docker ps

# Todos os containers (incluindo parados)
docker ps -a

# Formato personalizado
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"

# Filtrar por status
docker ps -f "status=exited"

# Mostrar apenas IDs
docker ps -q

# Último container criado
docker ps -l
```

### Criar e executar containers

```bash
# Criar e iniciar (modo detached)
docker run --name meu-app -d nginx

# Modo interativo com terminal
docker run -it ubuntu bash

# Com mapeamento de porta
docker run -d -p 8080:80 nginx

# Com variáveis de ambiente
docker run -d -e POSTGRES_PASSWORD=secret postgres:15

# Com volume montado
docker run -d -v meu-volume:/data redis

# Com bind mount
docker run -d -v $(pwd)/app:/app python:3.11

# Com limite de recursos
docker run -d --memory=512m --cpus=1.5 nginx

# Com nome de host
docker run -d --hostname meu-host nginx

# Com política de reinício
docker run -d --restart=unless-stopped nginx

# Remover automaticamente ao parar
docker run --rm -it alpine sh

# Com rede específica
docker run -d --network minha-rede nginx

# Criar sem iniciar
docker create --name meu-container nginx
```

### Gerenciar containers

```bash
# Parar (SIGTERM seguido de SIGKILL após timeout)
docker stop meu-container

# Parar com timeout customizado (segundos)
docker stop -t 30 meu-container

# Parar todos os containers em execução
docker stop $(docker ps -q)

# Iniciar container parado
docker start meu-container

# Reiniciar
docker restart meu-container

# Pausar processos (freeze via cgroups)
docker pause meu-container
docker unpause meu-container

# Enviar sinal específico
docker kill --signal=SIGHUP meu-container

# Forçar parada (SIGKILL)
docker kill meu-container

# Remover container parado
docker rm meu-container

# Forçar remoção (mesmo se em execução)
docker rm -f meu-container

# Remover todos os containers parados
docker rm $(docker ps -aq -f "status=exited")

# Renomear container
docker rename nome-antigo nome-novo
```

### Executar comandos em containers

```bash
# Shell interativo
docker exec -it meu-container bash

# Shell se não houver bash
docker exec -it meu-container sh

# Comando único
docker exec meu-container cat /etc/hosts

# Como usuário específico
docker exec -u root meu-container whoami

# Com variáveis de ambiente
docker exec -e MY_VAR=value meu-container env

# Em diretório específico
docker exec -w /app meu-container ls
```

### Copiar arquivos

```bash
# Do host para o container
docker cp arquivo.txt meu-container:/app/arquivo.txt

# Do container para o host
docker cp meu-container:/app/logs/ ./logs-backup/

# Copiar com preservação de permissões
docker cp --archive meu-container:/data ./data-backup
```

### Aguardar container encerrar

```bash
# Bloquear até o container parar e retornar o exit code
docker wait meu-container
```

---

## <a id="imagens">Comandos Essenciais — Imagens</a>

### Listar e buscar imagens

```bash
# Listar imagens locais
docker images

# Listar com filtro
docker images --filter "dangling=true"

# Listar apenas IDs
docker images -q

# Buscar no Docker Hub
docker search nginx

# Buscar com filtro de estrelas
docker search --filter stars=50 python
```

### Baixar e publicar imagens

```bash
# Baixar imagem (tag padrão: latest)
docker pull nginx

# Baixar tag específica
docker pull python:3.11-slim

# Baixar por digest
docker pull nginx@sha256:abc123...

# Baixar todas as tags
docker pull -a nginx

# Publicar imagem
docker push meu-registry/minha-imagem:1.0
```

### Construir imagens

```bash
# Build básico
docker build -t minha-imagem .

# Com tag e versão
docker build -t minha-imagem:1.0 .

# Múltiplas tags
docker build -t minha-imagem:1.0 -t minha-imagem:latest .

# Com build args
docker build --build-arg VERSION=3.11 -t app .

# Sem usar cache
docker build --no-cache -t app .

# De um Dockerfile diferente
docker build -f Dockerfile.prod -t app:prod .

# Build com target específico (multi-stage)
docker build --target builder -t app:build .

# Build com contexto remoto (Git)
docker build https://github.com/user/repo.git#main -t app .
```

### Tag e gerenciamento

```bash
# Criar tag
docker tag imagem-original:latest meu-registry/imagem:1.0

# Remover imagem
docker rmi nginx:latest

# Forçar remoção
docker rmi -f imagem

# Histórico de camadas
docker history minha-imagem

# Salvar imagem como arquivo tar
docker save -o backup.tar minha-imagem:1.0

# Carregar imagem de arquivo tar
docker load -i backup.tar

# Criar imagem a partir de container em execução
docker commit meu-container minha-imagem-custom:1.0

# Exportar filesystem do container
docker export meu-container > container-fs.tar

# Importar filesystem como imagem
docker import container-fs.tar minha-imagem:import
```

---

## <a id="inspecionar-e-depurar">Inspecionar e Depurar</a>

### Inspecionar containers

```bash
# Informações detalhadas em JSON
docker inspect meu-container

# IP do container
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' meu-container

# Status do container
docker inspect -f '{{.State.Status}}' meu-container

# Variáveis de ambiente
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' meu-container

# Volumes montados
docker inspect -f '{{json .Mounts}}' meu-container | python -m json.tool

# Port bindings
docker inspect -f '{{json .NetworkSettings.Ports}}' meu-container
```

### Monitorar recursos

```bash
# Estatísticas em tempo real de todos os containers
docker stats

# De um container específico
docker stats meu-container

# Snapshot único (sem streaming)
docker stats --no-stream

# Formato personalizado
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Processos dentro do container
docker top meu-container

# Mudanças no filesystem do container
docker diff meu-container
```

### Depuração de containers

```bash
# Acessar container com problemas (override de entrypoint)
docker run -it --entrypoint sh minha-imagem

# Inspecionar container parado (criar novo a partir do estado)
docker commit container-parado debug-img
docker run -it debug-img sh

# Ver eventos do Docker em tempo real
docker events

# Ver eventos filtrados
docker events --filter 'type=container' --filter 'event=die'

# Sistema de informações completo
docker info

# Uso de disco detalhado
docker system df
docker system df -v
```

---

## <a id="logs">Logs</a>

```bash
# Ver logs do container
docker logs meu-container

# Seguir logs em tempo real
docker logs -f meu-container

# Últimas N linhas
docker logs --tail 100 meu-container

# Desde um horário específico
docker logs --since "2025-01-01T00:00:00" meu-container

# Últimos 30 minutos
docker logs --since 30m meu-container

# Com timestamps
docker logs -t meu-container

# Combinar: últimas 50 linhas + follow + timestamps
docker logs --tail 50 -tf meu-container
```

---

## <a id="dockerfile-referencia-completa">Dockerfile — Referência Completa</a>

### Estrutura e instruções

#### `FROM` — Imagem base

```dockerfile
# Imagem base simples
FROM ubuntu:22.04

# Com alias para multi-stage
FROM python:3.11-slim AS builder

# Imagem "vazia" (para binários estáticos)
FROM scratch
```

#### `WORKDIR` — Diretório de trabalho

```dockerfile
# Define diretório (cria se não existir)
WORKDIR /app

# Múltiplos WORKDIRs se combinam
WORKDIR /app
WORKDIR src
# Resultado: /app/src
```

#### `COPY` e `ADD`

```dockerfile
# Copiar arquivo
COPY requirements.txt .

# Copiar diretório
COPY src/ ./src/

# Copiar com permissões específicas
COPY --chmod=755 script.sh /usr/local/bin/

# Copiar com owner específico
COPY --chown=appuser:appuser . /app/

# ADD extrai tarballs automaticamente e aceita URLs (evite; prefira COPY)
ADD app.tar.gz /app/
```

#### `RUN` — Executar comandos durante o build

```dockerfile
# Forma shell (executa em /bin/sh -c)
RUN apt-get update && apt-get install -y curl

# Forma exec (sem shell)
RUN ["apt-get", "install", "-y", "curl"]

# Agrupar comandos para reduzir camadas
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Com mount de cache (BuildKit)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Com mount de secrets (BuildKit)
RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
    aws s3 cp s3://bucket/file .
```

#### `CMD` — Comando padrão

```dockerfile
# Forma exec (recomendada)
CMD ["python", "app.py"]

# Forma shell
CMD python app.py

# Como parâmetros para ENTRYPOINT
CMD ["--help"]
```

#### `ENTRYPOINT` — Ponto de entrada

```dockerfile
# Forma exec (recomendada)
ENTRYPOINT ["python", "app.py"]

# Combinação ENTRYPOINT + CMD
ENTRYPOINT ["python"]
CMD ["app.py"]
# Resultado: python app.py
# Override: docker run imagem outro.py → python outro.py
```

**Diferença entre CMD e ENTRYPOINT:**

| Aspecto             | CMD                                | ENTRYPOINT                          |
|---------------------|------------------------------------|-------------------------------------|
| Override no `run`   | Substituído inteiramente           | Argumentos são adicionados ao final |
| Uso típico          | Comando padrão substituível        | Executável fixo da imagem           |
| override flag       | argumentos no `docker run`         | `--entrypoint`                      |

#### `ENV` — Variáveis de ambiente

```dockerfile
# Variável única
ENV APP_ENV=production

# Múltiplas variáveis
ENV APP_ENV=production \
    APP_PORT=8000 \
    APP_DEBUG=false

# Disponível em RUN, CMD e no container em execução
```

#### `ARG` — Argumentos de build

```dockerfile
# Declarar argumento
ARG PYTHON_VERSION=3.11

# Usar no build
FROM python:${PYTHON_VERSION}-slim

# ARG antes de FROM vale para FROM; após FROM precisa ser redeclarado
ARG PYTHON_VERSION
ARG BUILD_DATE
LABEL build-date=$BUILD_DATE

# Passar no build: docker build --build-arg PYTHON_VERSION=3.12 .
```

**Diferença entre ENV e ARG:**

| Aspecto              | ENV                         | ARG                             |
|----------------------|-----------------------------|---------------------------------|
| Disponível no build  | Sim                         | Sim                             |
| Disponível no runtime| Sim                         | Não                             |
| Override             | `docker run -e`             | `docker build --build-arg`      |
| Persiste na imagem   | Sim (como var. de ambiente) | Não                             |

#### `EXPOSE`

```dockerfile
# Declarar porta (documentação; não publica automaticamente)
EXPOSE 8080
EXPOSE 8080/tcp 9090/udp
```

#### `VOLUME`

```dockerfile
# Declarar ponto de montagem
VOLUME /data
VOLUME ["/data", "/logs"]
```

#### `USER`

```dockerfile
# Criar e usar usuário não-root
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

# Por UID/GID
USER 1000:1000
```

#### `LABEL`

```dockerfile
LABEL maintainer="equipe@empresa.com"
LABEL version="1.0"
LABEL description="Descrição da imagem"

# OCI standard labels
LABEL org.opencontainers.image.source="https://github.com/user/repo"
LABEL org.opencontainers.image.created="2025-01-01"
```

#### `SHELL`

```dockerfile
# Alterar shell padrão (útil no Windows ou para usar bash)
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN curl -fsSL https://example.com | bash
```

#### `STOPSIGNAL`

```dockerfile
# Sinal enviado ao parar o container (padrão: SIGTERM)
STOPSIGNAL SIGQUIT
```

#### `ONBUILD`

```dockerfile
# Triggers executados quando esta imagem é usada como base
ONBUILD COPY . /app
ONBUILD RUN pip install -r requirements.txt
```

### Exemplo completo de Dockerfile de produção

```dockerfile
# syntax=docker/dockerfile:1
ARG PYTHON_VERSION=3.11

# ---- Estágio de build ----
FROM python:${PYTHON_VERSION}-slim AS builder

WORKDIR /app

# Instalar dependências de sistema para build
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Copiar e instalar dependências Python
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-compile --prefix=/install -r requirements.txt

# ---- Estágio de runtime ----
FROM python:${PYTHON_VERSION}-slim AS runtime

# Metadados
LABEL maintainer="equipe@empresa.com" \
      version="1.0.0" \
      org.opencontainers.image.source="https://github.com/user/app"

# Instalar dependências de runtime apenas
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

# Criar usuário não-root
RUN groupadd -r appuser && useradd -r -g appuser -d /app appuser

# Copiar dependências instaladas
COPY --from=builder /install /usr/local

# Copiar código
WORKDIR /app
COPY --chown=appuser:appuser . .

# Variáveis de ambiente
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    APP_ENV=production

# Porta
EXPOSE 8000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Usuário não-root
USER appuser

# Entrypoint
ENTRYPOINT ["python", "-m", "gunicorn"]
CMD ["app:app", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

---

## <a id="dockerignore">.dockerignore</a>

Arquivo que exclui arquivos/diretórios do contexto de build, reduzindo tamanho e tempo.

```dockerignore
# Controle de versão
.git
.gitignore

# Ambientes virtuais
.venv/
venv/
__pycache__/
*.pyc

# IDE
.vscode/
.idea/

# Dependências
node_modules/

# Build e artefatos
dist/
build/
*.egg-info/

# Docker
docker-compose*.yml
Dockerfile*
.dockerignore

# Arquivos de configuração local
.env
.env.local
*.log

# Documentação
README.md
docs/
LICENSE

# Testes (se não forem incluídos na imagem)
tests/
*.test.js
```

---

## <a id="multi-stage-builds">Multi-stage Builds</a>

Multi-stage builds permitem separar etapas de compilação/build do runtime final, resultando em imagens menores e mais seguras.

### Conceito

```dockerfile
# Stage 1: Build — imagem grande com ferramentas
FROM golang:1.21 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app .

# Stage 2: Runtime — imagem mínima
FROM alpine:3.19
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

### Python com multi-stage

```dockerfile
FROM python:3.11-slim AS build
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

### Node.js com multi-stage

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY package*.json ./
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Copiar de imagem externa

```dockerfile
# Copiar binários de outras imagens
COPY --from=nginx:alpine /etc/nginx/nginx.conf /etc/nginx/
```

### Build de target específico

```bash
# Construir apenas até o estágio "builder"
docker build --target builder -t app:build-stage .
```

---

## <a id="camadas-e-cache">Camadas e Cache</a>

### Como funciona o cache

Cada instrução `RUN`, `COPY`, `ADD` cria uma camada. Docker reutiliza camadas em cache se:

1. A instrução não mudou.
2. Os arquivos copiados (para `COPY`/`ADD`) não mudaram.
3. Todas as camadas anteriores estão em cache.

### Otimizar uso de cache

```dockerfile
# ✅ BOM: dependências primeiro (mudam menos), código depois
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

# ❌ RUIM: qualquer mudança no código invalida o cache de instalação
COPY . .
RUN pip install -r requirements.txt
```

### Cache em CI/CD

```bash
# Usar cache de builds anteriores
docker build --cache-from registry/app:cache -t app .

# Export/import de cache com BuildKit
docker buildx build \
  --cache-to type=registry,ref=registry/app:cache \
  --cache-from type=registry,ref=registry/app:cache \
  -t app .

# Cache local
docker buildx build \
  --cache-to type=local,dest=./cache \
  --cache-from type=local,src=./cache \
  -t app .
```

### Inline cache

```bash
# Embutir metadados de cache na imagem para reutilização
DOCKER_BUILDKIT=1 docker build --build-arg BUILDKIT_INLINE_CACHE=1 -t app .
```

---

## <a id="buildkit-e-docker-buildx">BuildKit e `docker buildx`</a>

BuildKit é o backend de build moderno do Docker com builds paralelos, cache avançado e suporte a secrets.

### Habilitar BuildKit

```bash
# Via variável de ambiente
export DOCKER_BUILDKIT=1

# Ou permanentemente em /etc/docker/daemon.json
{
  "features": { "buildkit": true }
}
```

### Funcionalidades exclusivas do BuildKit

```dockerfile
# Mount de cache (acelera installs repetidos)
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Mount de secrets (não persiste na imagem)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install

# Mount de SSH
RUN --mount=type=ssh \
    git clone git@github.com:user/repo.git

# Heredoc (inline files)
COPY <<EOF /app/config.yaml
database:
  host: localhost
  port: 5432
EOF
```

### Buildx — Comandos

```bash
# Criar builder
docker buildx create --name meu-builder --use

# Listar builders
docker buildx ls

# Inspecionar builder
docker buildx inspect meu-builder

# Build e carregar localmente
docker buildx build --load -t app .

# Build e push direto ao registry
docker buildx build --push -t registry/app:1.0 .

# Remover builder
docker buildx rm meu-builder
```

---

## <a id="builds-multi-plataforma">Builds Multi-plataforma</a>

Construir imagens para múltiplas arquiteturas (amd64, arm64, etc.) em uma única operação.

```bash
# Criar builder com suporte multi-plataforma
docker buildx create --name multiarch --driver docker-container --use

# Build para múltiplas plataformas
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t registry/app:1.0 .

# Inspecionar manifesto multi-plataforma
docker manifest inspect registry/app:1.0
```

```dockerfile
# No Dockerfile, use ARGs automáticos para detectar plataforma
ARG TARGETPLATFORM
ARG TARGETARCH
RUN echo "Building for ${TARGETARCH}"
```

---

## <a id="reducao-de-tamanho-de-imagem">Redução de Tamanho de Imagem</a>

### Escolha da imagem base

| Imagem                | Tamanho aprox. | Quando usar                            |
|-----------------------|----------------|----------------------------------------|
| `ubuntu:22.04`        | ~77 MB         | Quando precisa de glibc e ferramentas  |
| `debian:bookworm-slim`| ~74 MB         | Base geral com glibc, sem extras       |
| `python:3.11-slim`    | ~125 MB        | Python sem ferramentas de build        |
| `alpine:3.19`         | ~7 MB          | Imagem mínima com musl libc            |
| `distroless`          | ~20 MB         | Apenas runtime, sem shell              |
| `scratch`             | 0 MB           | Para binários estáticos (Go, Rust)     |

### Técnicas de redução

```dockerfile
# 1. Agrupar RUN e limpar cache no mesmo layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends pkg && \
    rm -rf /var/lib/apt/lists/*

# 2. Não instalar pacotes recomendados
RUN apt-get install -y --no-install-recommends curl

# 3. Multi-stage: copiar apenas artefatos necessários
COPY --from=builder /app/bin /usr/local/bin/

# 4. Usar .dockerignore para reduzir contexto de build

# 5. Compilar Python sem bytecode
ENV PYTHONDONTWRITEBYTECODE=1

# 6. Remover docs e man pages
RUN rm -rf /usr/share/doc /usr/share/man

# 7. Usar imagens distroless para produção
FROM gcr.io/distroless/python3-debian12
```

### Analisar tamanho de imagem

```bash
# Ver camadas e tamanhos
docker history minha-imagem

# Ferramenta dive (análise interativa de layers)
dive minha-imagem
```

---

## <a id="volumes-e-persistencia-de-dados">Volumes e Persistência de Dados</a>

### Tipos de armazenamento

| Tipo         | Gerenciado pelo Docker | Uso                               |
|--------------|------------------------|-----------------------------------|
| Volume       | Sim                    | Dados persistentes em produção    |
| Bind mount   | Não                    | Desenvolvimento local             |
| tmpfs        | Não (em memória)       | Dados temporários e secretos      |

### Volumes nomeados

```bash
# Criar volume
docker volume create meu-volume

# Listar volumes
docker volume ls

# Inspecionar volume
docker volume inspect meu-volume

# Usar volume no container
docker run -d -v meu-volume:/data redis

# Sintaxe --mount (mais explícita)
docker run -d --mount source=meu-volume,target=/data redis

# Volume somente leitura
docker run -d -v meu-volume:/data:ro redis

# Remover volume
docker volume rm meu-volume

# Remover volumes não utilizados
docker volume prune

# Volume com driver específico (NFS, cloud, etc.)
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/shared \
  nfs-volume
```

### Backup e restauração de volumes

```bash
# Backup
docker run --rm \
  -v meu-volume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/meu-volume-backup.tar.gz -C /data .

# Restauração
docker run --rm \
  -v meu-volume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/meu-volume-backup.tar.gz -C /data
```

---

## <a id="bind-mounts-e-tmpfs">Bind Mounts e tmpfs</a>

### Bind mounts

```bash
# Montar diretório local no container
docker run -d -v $(pwd)/app:/app python:3.11

# Sintaxe --mount
docker run -d --mount type=bind,source=$(pwd)/app,target=/app python:3.11

# Bind mount somente leitura
docker run -d -v $(pwd)/config:/config:ro nginx

# Montar arquivo individual
docker run -d -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro nginx
```

### tmpfs (armazenamento em memória)

```bash
# Montar tmpfs
docker run -d --tmpfs /tmp:rw,noexec,nosuid,size=100m nginx

# Sintaxe --mount
docker run -d --mount type=tmpfs,destination=/tmp,tmpfs-size=100m nginx
```

---

## <a id="networking">Networking</a>

### Drivers de rede

| Driver    | Descrição                                     | Uso                              |
|-----------|-----------------------------------------------|----------------------------------|
| bridge    | Rede isolada padrão no host                   | Containers no mesmo host         |
| host      | Compartilha rede do host                      | Performance máxima               |
| none      | Sem rede                                      | Containers isolados              |
| overlay   | Rede entre múltiplos hosts (Swarm)            | Clusters                         |
| macvlan   | Atribui MAC address ao container              | Integração com rede física       |
| ipvlan    | Similar ao macvlan, compartilha MAC do host   | Ambientes com restrições de MAC  |

### Gerenciar redes

```bash
# Criar rede bridge customizada
docker network create minha-rede

# Criar com subnet específica
docker network create --subnet=172.20.0.0/16 --gateway=172.20.0.1 minha-rede

# Criar rede com driver específico
docker network create --driver overlay minha-overlay

# Listar redes
docker network ls

# Inspecionar rede
docker network inspect minha-rede

# Conectar container a uma rede
docker network connect minha-rede meu-container

# Conectar com IP fixo
docker network connect --ip 172.20.0.10 minha-rede meu-container

# Desconectar container de uma rede
docker network disconnect minha-rede meu-container

# Remover rede
docker network rm minha-rede

# Remover redes não utilizadas
docker network prune
```

### Comunicação entre containers

```bash
# Containers na mesma rede podem se comunicar pelo nome
docker network create app-net
docker run -d --name db --network app-net postgres:15
docker run -d --name web --network app-net nginx

# De dentro do container "web":
# ping db → resolve para o IP do container "db"
# psql -h db → conecta ao PostgreSQL
```

### Rede host

```bash
# Compartilhar stack de rede do host (sem isolamento)
docker run -d --network host nginx
# nginx escutará diretamente na porta 80 do host
```

### DNS customizado

```bash
# Usar DNS específico
docker run -d --dns 8.8.8.8 --dns 8.8.4.4 nginx

# Alias de rede (múltiplos nomes para o mesmo container)
docker network connect --alias api --alias backend minha-rede meu-container
```

---

## <a id="mapeamento-de-portas">Mapeamento de Portas</a>

```bash
# Porta do host:porta do container
docker run -d -p 8080:80 nginx

# Bind em interface específica
docker run -d -p 127.0.0.1:8080:80 nginx

# Porta aleatória do host
docker run -d -p 80 nginx

# Múltiplas portas
docker run -d -p 8080:80 -p 8443:443 nginx

# Range de portas
docker run -d -p 8000-8010:8000-8010 app

# Publicar todas as portas declaradas com EXPOSE
docker run -d -P nginx

# UDP
docker run -d -p 5353:53/udp dns-server

# Verificar mapeamento de portas
docker port meu-container
```

---

## <a id="variaveis-de-ambiente">Variáveis de Ambiente</a>

```bash
# Variável individual
docker run -d -e DATABASE_URL=postgres://user:pass@db/app nginx

# Múltiplas variáveis
docker run -d \
  -e APP_ENV=production \
  -e APP_DEBUG=false \
  -e SECRET_KEY=abc123 \
  app

# Herdar variável do host
docker run -d -e HOME app

# Arquivo de variáveis (.env)
docker run -d --env-file .env app
```

Formato do arquivo `.env`:

```ini
# Comentários são permitidos
DATABASE_URL=postgres://user:pass@db/app
APP_ENV=production
APP_DEBUG=false
SECRET_KEY=minha-chave-secreta
```

---

## <a id="limites-de-recursos">Limites de Recursos</a>

### Memória

```bash
# Limite de memória
docker run -d --memory=512m app

# Limite de memória + swap
docker run -d --memory=512m --memory-swap=1g app

# Reserva de memória (soft limit)
docker run -d --memory=1g --memory-reservation=512m app

# Desabilitar OOM killer
docker run -d --memory=512m --oom-kill-disable app
```

### CPU

```bash
# Limitar número de CPUs
docker run -d --cpus=1.5 app

# CPU shares (peso relativo, padrão: 1024)
docker run -d --cpu-shares=512 app

# Fixar em CPUs específicas
docker run -d --cpuset-cpus="0,2" app

# Fixar em range de CPUs
docker run -d --cpuset-cpus="0-3" app
```

### I/O

```bash
# Limitar I/O de disco (bytes/s)
docker run -d --device-write-bps /dev/sda:10mb app

# Limitar IOPS
docker run -d --device-write-iops /dev/sda:100 app

# Peso relativo de I/O (10-1000, padrão 500)
docker run -d --blkio-weight 300 app
```

### PIDs

```bash
# Limitar número de processos
docker run -d --pids-limit 100 app
```

### Verificar limites aplicados

```bash
docker inspect -f '{{.HostConfig.Memory}}' meu-container
docker stats --no-stream
```

---

## <a id="healthchecks">Healthchecks</a>

### No Dockerfile

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

| Parâmetro        | Descrição                                         | Padrão |
|------------------|----------------------------------------------------|--------|
| `--interval`     | Intervalo entre verificações                       | 30s    |
| `--timeout`      | Tempo máximo de espera por resposta                | 30s    |
| `--start-period` | Período de grace após start (falhas não contam)    | 0s     |
| `--retries`      | Tentativas antes de marcar como unhealthy          | 3      |

### No `docker run`

```bash
docker run -d \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 30s \
  --health-timeout 5s \
  --health-start-period 10s \
  --health-retries 3 \
  app
```

### Desabilitar healthcheck

```bash
docker run -d --no-healthcheck app
```

### Verificar status de saúde

```bash
docker inspect -f '{{.State.Health.Status}}' meu-container
docker inspect -f '{{json .State.Health}}' meu-container | python -m json.tool
```

### Exemplos de healthchecks por serviço

```dockerfile
# PostgreSQL
HEALTHCHECK CMD pg_isready -U postgres || exit 1

# Redis
HEALTHCHECK CMD redis-cli ping | grep PONG || exit 1

# HTTP genérico (sem curl instalado)
HEALTHCHECK CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# TCP check
HEALTHCHECK CMD bash -c 'cat < /dev/null > /dev/tcp/localhost/5432' || exit 1
```

---

## <a id="seguranca">Segurança</a>

### Usuário não-root

```dockerfile
# Criar usuário e grupo
RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser

# Definir ownership dos arquivos
COPY --chown=appuser:appuser . /app/

# Mudar para o usuário
USER appuser
```

```bash
# Override no runtime
docker run -u 1000:1000 app
```

### Capabilities

```bash
# Remover todas as capabilities e adicionar apenas as necessárias
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE app

# Listar capabilities de um container
docker inspect -f '{{.HostConfig.CapAdd}}' meu-container
docker inspect -f '{{.HostConfig.CapDrop}}' meu-container
```

### Read-only filesystem

```bash
# Filesystem somente leitura
docker run --read-only app

# Com tmpfs para diretórios que precisam de escrita
docker run --read-only --tmpfs /tmp --tmpfs /run app
```

### Sem novos privilégios

```bash
# Impedir escalação de privilégios (setuid, setgid)
docker run --security-opt no-new-privileges app
```

### Seccomp, AppArmor e SELinux

```bash
# Perfil seccomp customizado
docker run --security-opt seccomp=perfil.json app

# Perfil AppArmor
docker run --security-opt apparmor=docker-default app

# Label SELinux
docker run --security-opt label=type:container_t app
```

### Docker Content Trust (DCT)

```bash
# Habilitar assinatura e verificação de imagens
export DOCKER_CONTENT_TRUST=1

# Push assinado
docker push registry/app:1.0

# Pull verificado (rejeita imagens não assinadas)
docker pull registry/app:1.0
```

### Cosign (sigstore)

```bash
# Assinar imagem
cosign sign registry/app:1.0

# Verificar assinatura
cosign verify registry/app:1.0
```

### Escaneamento de vulnerabilidades

```bash
# Trivy
trivy image minha-imagem:1.0

# Trivy com severidade mínima
trivy image --severity HIGH,CRITICAL minha-imagem:1.0

# Docker Scout (integrado)
docker scout cves minha-imagem:1.0
docker scout recommendations minha-imagem:1.0

# Grype
grype minha-imagem:1.0

# Snyk
snyk container test minha-imagem:1.0
```

### Secrets em runtime

```bash
# Docker Secrets (Swarm mode)
echo "minha-senha" | docker secret create db_password -
docker service create --secret db_password app

# Sem Swarm: montar secrets como arquivos
docker run -v ./secret.txt:/run/secrets/secret:ro app

# BuildKit secrets (não persiste na imagem)
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

### Boas práticas de segurança — Resumo

1. Use imagens oficiais e mínimas.
2. Sempre execute como usuário não-root.
3. Escaneie imagens no pipeline de CI.
4. Use `--cap-drop=ALL` e adicione apenas capabilities necessárias.
5. Ative `--read-only` quando possível.
6. Nunca coloque segredos no Dockerfile ou na imagem.
7. Use tags fixas ou digests, nunca `latest` em produção.
8. Mantenha imagens atualizadas (rebuild periódico).
9. Limite recursos (memória, CPU, PIDs).
10. Use Docker Content Trust ou cosign para verificar imagens.

---

## <a id="docker-registry">Docker Registry</a>

### Docker Hub

```bash
# Login
docker login

# Login em registry privado
docker login registry.empresa.com

# Logout
docker logout

# Tag para publicação
docker tag app:1.0 usuario/app:1.0

# Push
docker push usuario/app:1.0

# Pull
docker pull usuario/app:1.0
```

### Registry privado local

```bash
# Subir registry local
docker run -d -p 5000:5000 --name registry \
  -v registry-data:/var/lib/registry \
  registry:2

# Tag e push para registry local
docker tag app:1.0 localhost:5000/app:1.0
docker push localhost:5000/app:1.0

# Pull do registry local
docker pull localhost:5000/app:1.0

# Listar repositórios (API)
curl http://localhost:5000/v2/_catalog

# Listar tags de um repositório
curl http://localhost:5000/v2/app/tags/list
```

### Registries de nuvem

```bash
# AWS ECR
aws ecr get-login-password | docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/app:1.0

# Google GCR / Artifact Registry
gcloud auth configure-docker
docker push gcr.io/meu-projeto/app:1.0

# Azure ACR
az acr login --name meuregistry
docker push meuregistry.azurecr.io/app:1.0

# GitHub Container Registry
echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin
docker push ghcr.io/user/app:1.0
```

### Estratégia de tags

```bash
# Tag semântica (recomendada para produção)
docker tag app:latest registry/app:1.2.3
docker tag app:latest registry/app:1.2
docker tag app:latest registry/app:1

# Tag por commit SHA
docker tag app:latest registry/app:$(git rev-parse --short HEAD)

# Tag por digest (imutável)
docker pull registry/app@sha256:abc123...
```

---

## <a id="limpeza-e-manutencao">Limpeza e Manutenção</a>

```bash
# Remover containers parados, redes não usadas, imagens dangling e cache de build
docker system prune

# Incluir volumes não utilizados
docker system prune --volumes

# Incluir todas as imagens não usadas (não apenas dangling)
docker system prune -a

# Limpeza forçada (sem confirmação)
docker system prune -af --volumes

# Uso de disco detalhado
docker system df
docker system df -v

# Limpeza por tipo
docker container prune    # containers parados
docker image prune        # imagens dangling
docker image prune -a     # todas as imagens não usadas
docker volume prune       # volumes órfãos
docker network prune      # redes não usadas
docker builder prune      # cache de build

# Remover imagens por filtro
docker image prune -a --filter "until=720h"  # imagens com mais de 30 dias

# Remover containers por filtro
docker container prune --filter "until=24h"
```

---

## <a id="observabilidade-e-logs-em-producao">Observabilidade e Logs em Produção</a>

### Logging drivers

```bash
# Usar driver de log específico
docker run -d --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 app

# Listar drivers disponíveis
docker info --format '{{.Plugins.Log}}'
```

| Driver        | Descrição                            |
|---------------|--------------------------------------|
| `json-file`   | JSON local (padrão)                  |
| `syslog`      | Syslog do host                       |
| `journald`    | systemd journal                      |
| `fluentd`     | Forward para Fluentd                 |
| `awslogs`     | AWS CloudWatch                       |
| `gcplogs`     | Google Cloud Logging                 |
| `splunk`      | Splunk HTTP Event Collector          |
| `none`        | Desabilita logs                      |

### Configurar logging global

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5",
    "labels": "app,environment",
    "tag": "{{.ImageName}}/{{.Name}}/{{.ID}}"
  }
}
```

### Boas práticas de logging

- Redirecione logs para `stdout`/`stderr` no container.
- Agregue com **Fluentd/Fluent Bit/Logstash/Vector**.
- Centralize em **Elasticsearch, Loki ou CloudWatch**.
- Use labels/tags para identificar serviços nos logs.

### Métricas e monitoramento

```bash
# Habilitar endpoint de métricas do Docker daemon
# Em /etc/docker/daemon.json:
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}

# Métricas via cAdvisor (container-level)
docker run -d \
  --name cadvisor \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro \
  -p 8080:8080 \
  gcr.io/cadvisor/cadvisor
```

- **cAdvisor:** métricas de CPU, memória, I/O, rede por container.
- **Prometheus:** coleta métricas de cAdvisor e do daemon.
- **Grafana:** dashboards e alertas.
- **Datadog, New Relic:** monitoramento gerenciado.

---

## <a id="cicd-com-docker">CI/CD com Docker</a>

### Boas práticas para pipelines

```bash
# Build reprodutível com tag de commit
docker build -t registry/app:$(git rev-parse --short HEAD) .

# Push com múltiplas tags
docker tag registry/app:$SHA registry/app:$BRANCH
docker tag registry/app:$SHA registry/app:latest
docker push registry/app:$SHA
docker push registry/app:$BRANCH

# Teste de integração com compose
docker compose -f docker-compose.ci.yml up --build --abort-on-container-exit
EXIT_CODE=$?
docker compose -f docker-compose.ci.yml down --volumes
exit $EXIT_CODE
```

### GitHub Actions — Exemplo

```yaml
name: Build and Push
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### GitLab CI — Exemplo

```yaml
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

### Scanning no pipeline

```yaml
# Exemplo com Trivy em GitHub Actions
- name: Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: registry/app:${{ github.sha }}
    format: table
    exit-code: 1
    severity: CRITICAL,HIGH
```

---

## <a id="docker-em-producao-boas-praticas">Docker em Produção — Boas Práticas</a>

### Checklist de produção

- [ ] Imagens mínimas e multi-stage.
- [ ] Usuário não-root.
- [ ] Tags imutáveis (sem `latest`).
- [ ] Healthchecks configurados.
- [ ] Limites de recursos (memória, CPU).
- [ ] Logging para stdout/stderr com driver adequado.
- [ ] Rotação de logs configurada.
- [ ] Secrets injetados via runtime, nunca na imagem.
- [ ] Escaneamento de vulnerabilidades no CI.
- [ ] `.dockerignore` atualizado.
- [ ] Restart policy (`unless-stopped` ou `always`).
- [ ] Rede bridge customizada (não usar default).
- [ ] `--read-only` quando possível.
- [ ] Capabilities mínimas.
- [ ] Backups regulares de volumes.

### Restart policies

```bash
# Nunca reiniciar (padrão)
docker run --restart=no app

# Sempre reiniciar (exceto docker stop)
docker run --restart=always app

# Reiniciar se o processo falhar
docker run --restart=on-failure app

# Reiniciar com limite de tentativas
docker run --restart=on-failure:5 app

# Reiniciar a menos que manualmente parado
docker run --restart=unless-stopped app
```

### Runtime e compatibilidade

- Teste imagens em runtimes distintos (Docker Engine, containerd).
- Em ambientes com **SELinux**, adicione `:z` ou `:Z` aos bind mounts.
- Em ambientes com **AppArmor**, crie perfis personalizados.

### Live restore

```json
// Containers continuam rodando mesmo se o daemon reiniciar
{
  "live-restore": true
}
```

---

## <a id="docker-contexts">Docker Contexts</a>

Contexts permitem gerenciar múltiplos Docker daemons (local, remoto, cloud).

```bash
# Listar contexts
docker context ls

# Criar context para host remoto
docker context create meu-server --docker "host=ssh://user@192.168.1.100"

# Usar context
docker context use meu-server

# Executar comando em contexto específico
docker --context meu-server ps

# Remover context
docker context rm meu-server

# Exportar/importar context
docker context export meu-server > backup.dockercontext
docker context import meu-server backup.dockercontext
```

---

## <a id="docker-compose-orquestracao-local">Docker Compose — Orquestração Local</a>

`docker compose` (plugin v2) ou `docker-compose` (v1) simplifica o gerenciamento de aplicações multi-container.

### Estrutura básica (`docker-compose.yml`)

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

### Comandos essenciais

```bash
# Subir serviços (detached)
docker compose up -d

# Subir com rebuild de imagens
docker compose up -d --build

# Subir serviço específico
docker compose up -d web

# Parar serviços
docker compose stop

# Parar e remover containers, redes
docker compose down

# Parar e remover tudo (volumes, imagens)
docker compose down --volumes --rmi all

# Ver status dos serviços
docker compose ps

# Ver logs
docker compose logs -f

# Logs de serviço específico
docker compose logs -f web

# Executar comando em serviço
docker compose exec web bash

# Executar comando one-off (novo container)
docker compose run --rm web python manage.py migrate

# Reconstruir imagens
docker compose build

# Reconstruir sem cache
docker compose build --no-cache

# Pull de imagens atualizadas
docker compose pull

# Ver configuração compilada
docker compose config

# Eventos em tempo real
docker compose events
```

### `depends_on` com condições

```yaml
services:
  web:
    build: .
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
```

### Variáveis de ambiente e `.env`

```yaml
services:
  web:
    image: app:${APP_VERSION:-latest}
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - APP_ENV=${APP_ENV:-development}
    env_file:
      - .env
      - .env.local
```

```ini
# .env
APP_VERSION=1.2.3
DATABASE_URL=postgres://user:pass@db:5432/app
APP_ENV=production
```

### Perfis (profiles)

```yaml
services:
  web:
    image: app
    profiles: []  # sempre ativo (sem profile)

  db:
    image: postgres:15
    profiles: []

  debug:
    image: busybox
    profiles: ["debug"]

  monitoring:
    image: prometheus
    profiles: ["monitoring"]
```

```bash
# Subir serviços com perfil específico
docker compose --profile monitoring up -d

# Múltiplos perfis
docker compose --profile monitoring --profile debug up -d
```

### Sobreposição de configurações (override)

```yaml
# docker-compose.yml (base)
services:
  web:
    image: app:1.0
    ports:
      - "8000:8000"

# docker-compose.override.yml (auto-carregado em dev)
services:
  web:
    build: .
    volumes:
      - .:/app
    environment:
      - DEBUG=true

# docker-compose.prod.yml (produção)
services:
  web:
    image: registry/app:1.0
    restart: always
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
```

```bash
# Dev (usa base + override automaticamente)
docker compose up

# Produção (especificar arquivos)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### Volumes e bind mounts

```yaml
services:
  web:
    volumes:
      # Bind mount (desenvolvimento)
      - ./app:/app
      # Volume nomeado
      - static_data:/app/static
      # Somente leitura
      - ./config:/config:ro
      # tmpfs
      - type: tmpfs
        target: /tmp

  db:
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
    driver: local
  static_data:
    # Volume externo (deve existir previamente)
    external: true
```

### Redes

```yaml
services:
  web:
    networks:
      - frontend
      - backend

  api:
    networks:
      - backend

  db:
    networks:
      backend:
        ipv4_address: 172.28.0.10

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

### Secrets

```yaml
services:
  web:
    secrets:
      - db_password
      - api_key
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    environment: API_KEY
```

### Deploy e limites de recursos

```yaml
services:
  web:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
```

### Escalonamento

```bash
# Escalar serviço
docker compose up -d --scale worker=5

# Ver réplicas
docker compose ps
```

### Init containers e one-off tasks

```yaml
services:
  migrate:
    image: app
    command: python manage.py migrate
    depends_on:
      db:
        condition: service_healthy
    profiles: ["setup"]

  web:
    image: app
    depends_on:
      db:
        condition: service_healthy
```

```bash
# Executar migração antes de subir
docker compose --profile setup run --rm migrate
docker compose up -d
```

### Docker Compose Watch (auto-rebuild)

```yaml
services:
  web:
    build: .
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: ./requirements.txt
```

```bash
docker compose watch
```

### Extensões e includes (Compose v2.20+)

```yaml
# Incluir outros arquivos de compose
include:
  - path: ./monitoring/docker-compose.yml
  - path: ./logging/docker-compose.yml

services:
  web:
    build: .
```

### Integração com CI

```bash
# Testes de integração
docker compose -f docker-compose.ci.yml up --build --abort-on-container-exit
EXIT_CODE=$?
docker compose -f docker-compose.ci.yml down --volumes
exit $EXIT_CODE
```

### Boas práticas do Compose

1. Versione arquivos de compose no repositório.
2. Documente modo de uso no `README.md`.
3. Use `.env` para variáveis locais e não comite no repositório.
4. Prefira tags específicas; evite `latest`.
5. Use `override.yml` para configurações de desenvolvimento.
6. Configure healthchecks e `depends_on` com condições.
7. Limpe recursos com `docker compose down --volumes --rmi local`.
8. Use perfis para serviços opcionais (debug, monitoring).
9. Separe configurações de dev, staging e produção em arquivos distintos.

---

## <a id="troubleshooting">Troubleshooting</a>

### Container não inicia

```bash
# Ver logs do container
docker logs meu-container

# Ver detalhes do estado
docker inspect -f '{{json .State}}' meu-container | python -m json.tool

# Testar entrypoint manualmente
docker run -it --entrypoint sh minha-imagem
```

### Container reiniciando em loop

```bash
# Ver exit code
docker inspect -f '{{.State.ExitCode}}' meu-container

# Exit codes comuns:
# 0 = sucesso / encerramento normal
# 1 = erro genérico da aplicação
# 137 = killed (SIGKILL) — geralmente OOM killer
# 139 = segfault (SIGSEGV)
# 143 = terminated (SIGTERM) — docker stop graceful

# Verificar se foi OOM killed
docker inspect -f '{{.State.OOMKilled}}' meu-container
```

### Problemas de rede

```bash
# Testar DNS de dentro do container
docker exec meu-container nslookup google.com

# Testar conectividade
docker exec meu-container ping outro-container

# Verificar configuração de rede
docker inspect -f '{{json .NetworkSettings}}' meu-container

# Verificar se containers estão na mesma rede
docker network inspect minha-rede
```

### Problemas de permissão

```bash
# Verificar usuário do container
docker exec meu-container whoami
docker exec meu-container id

# Verificar permissões de arquivo
docker exec meu-container ls -la /app

# Forçar uid/gid no bind mount
docker run -u $(id -u):$(id -g) -v $(pwd):/app app
```

### Disco cheio

```bash
# Ver uso de disco do Docker
docker system df

# Limpeza completa
docker system prune -af --volumes

# Limpeza de cache de build
docker builder prune -af
```

### Container consumindo muita memória

```bash
# Verificar consumo
docker stats meu-container

# Adicionar limite
docker update --memory=512m --memory-swap=1g meu-container
```

### Slow builds

```bash
# Verificar tamanho do contexto de build
du -sh .

# Verificar .dockerignore
cat .dockerignore

# Habilitar BuildKit
export DOCKER_BUILDKIT=1

# Usar cache com buildx
docker buildx build --cache-from type=local,src=./cache -t app .
```

### Verificar daemon

```bash
# Status do daemon
sudo systemctl status docker

# Logs do daemon
sudo journalctl -u docker -f

# Verificar socket
ls -la /var/run/docker.sock
```

---

## <a id="alternativas-ao-docker">Alternativas ao Docker</a>

| Ferramenta   | Descrição                                                       |
|--------------|-----------------------------------------------------------------|
| **Podman**   | Daemonless, rootless, compatível com CLI Docker                 |
| **Buildah**  | Build de imagens OCI sem daemon                                 |
| **Skopeo**   | Copiar e inspecionar imagens entre registries                   |
| **nerdctl**  | CLI compatível com Docker para containerd                       |
| **LXC/LXD**  | Containers de sistema (mais próximos de VMs leves)              |
| **Kata**     | Containers com isolamento de VM (microVM)                       |
| **Firecracker** | microVMs da AWS (usado no Lambda e Fargate)                 |

```bash
# Podman (drop-in replacement)
alias docker=podman
podman run --rm -it alpine sh
podman build -t app .
podman compose up -d

# Buildah (build sem daemon)
buildah bud -t app .
buildah push app registry/app:1.0
```

---

Referências: [Documentação oficial Docker](https://docs.docker.com/), [Dockerfile reference](https://docs.docker.com/engine/reference/builder/), [Docker Compose reference](https://docs.docker.com/compose/compose-file/), [Docker Security](https://docs.docker.com/engine/security/), [BuildKit](https://docs.docker.com/build/buildkit/), [Trivy](https://aquasecurity.github.io/trivy/), [Snyk Container](https://snyk.io/product/container-vulnerability-management/).
