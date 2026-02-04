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
