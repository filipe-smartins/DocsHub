# Linux - Comandos de Terminal

Uma referência rápida para os comandos de terminal Linux mais comuns.

## Índice

- [Navegação e Arquivos](#navegacao-e-arquivos)
- [Visualização e Edição](#visualizacao-e-edicao)
- [Permissões e Sistema](#permissoes-e-sistema)

## <a id="navegacao-e-arquivos">Navegação e Arquivos</a>

- **Ver diretório atual:**
  ```bash
  pwd
  ```
- **Listar arquivos:**
  ```bash
  ls -la
  ```
- **Mudar de diretório:**
  ```bash
  cd /caminho/do/diretorio
  ```
- **Criar pasta:**
  ```bash
  mkdir nome-da-pasta
  ```
- **Remover arquivo:**
  ```bash
  rm nome-arquivo
  ```
- **Remover pasta (recursivo):**
  ```bash
  rm -rf nome-da-pasta
  ```
- **Copiar arquivo:**
  ```bash
  cp origem destino
  ```
- **Mover ou Renomear:**
  ```bash
  mv antigo novo
  ```

## <a id="visualizacao-e-edicao">Visualização e Edição</a>

- **Ler conteúdo do arquivo:**
  ```bash
  cat arquivo.txt
  ```
- **Ler arquivo paginado:**
  ```bash
  less arquivo.txt
  ```
- **Ver as últimas linhas (log):**
  ```bash
  tail -f arquivo.log
  ```
- **Editar arquivo no terminal:**
  ```bash
  nano arquivo.txt
  # ou
  vi arquivo.txt
  ```

## <a id="permissoes-e-sistema">Permissões e Sistema</a>

- **Mudar permissões:**
  ```bash
  chmod +x script.sh
  ```
- **Mudar dono do arquivo:**
  ```bash
  sudo chown usuario:grupo arquivo
  ```
- **Ver processos rodando:**
  ```bash
  top
  # ou
  htop
  ```
- **Ver uso de disco:**
  ```bash
  df -h
  ```
- **Ver uso de memória:**
  ```bash
  free -m
  ```
