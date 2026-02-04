# Git - Comandos Essenciais

Dominar o Git é fundamental para o controle de versão. Aqui estão os comandos mais usados.

[Ir para Configurações](#configuracoes)

## Índice

- [Configurações](#configuracoes)
- [Início e Configuração](#inicio-e-configuracao)
- [Fluxo de Trabalho](#fluxo-de-trabalho)
- [Branches](#branches)
- [Desfazendo Alterações](#desfazendo-alteracoes)

## <a id="configuracoes">Configurações</a>


## <a id="inicio-e-configuracao">Início e Configuração</a>

- **Inicializar um repositório:**
  ```bash
  git init
  ```
- **Clonar um repositório existente:**
  ```bash
  git clone https://github.com/usuario/repositorio.git
  ```
- **Configurar usuário:**
  ```bash
  git config --global user.name "Seu Nome"
  git config --global user.email "seuemail@exemplo.com"
  ```

## <a id="fluxo-de-trabalho">Fluxo de Trabalho</a>

- **Verificar status:**
  ```bash
  git status
  ```
- **Adicionar arquivos para o commit:**
  ```bash
  git add .
  ```
- **Realizar um commit:**
  ```bash
  git commit -m "Mensagem descritiva"
  ```
- **Enviar alterações para o servidor:**
  ```bash
  git push origin nome-da-branch
  ```
- **Baixar atualizações do servidor:**
  ```bash
  git pull origin nome-da-branch
  ```

## <a id="branches">Branches</a>

- **Listar branches:**
  ```bash
  git branch
  ```
- **Criar uma nova branch:**
  ```bash
  git checkout -b nome-nova-branch
  ```
- **Mudar para uma branch existente:**
  ```bash
  git checkout nome-da-branch
  ```
- **Mesclar branches (Merge):**
  ```bash
  git merge nome-da-branch
  ```

## <a id="desfazendo-alteracoes">Desfazendo Alterações</a>

- **Desfazer alterações locais (não commitadas):**
  ```bash
  git checkout -- nome-arquivo
  ```
- **Remover arquivo do status de commit (unstaged):**
  ```bash
  git reset HEAD nome-arquivo
  ```
- **Ver histórico de commits:**
  ```bash
  git log --oneline
  ```
