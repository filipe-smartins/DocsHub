# SSH - Guia Completo

SSH (Secure Shell) é um protocolo de rede criptográfico para comunicação segura entre máquinas. Ele substitui protocolos inseguros como Telnet, rlogin e FTP, oferecendo autenticação forte e comunicação criptografada. Este guia cobre desde o uso básico até configurações avançadas.

## Índice

- [Conceitos fundamentais](#conceitos-fundamentais)
- [Instalação](#instalacao)
- [Gerar chaves SSH](#gerar-chaves-ssh)
- [Copiar chave para servidor](#copiar-chave-para-servidor)
- [Conectar ao servidor](#conectar-ao-servidor)
- [Executar comandos remotos](#executar-comandos-remotos)
- [Arquivo de configuração do cliente](#arquivo-de-configuracao-do-cliente)
- [Agente SSH](#agente-ssh)
- [Transferência de arquivos (SCP e SFTP)](#transferencia-de-arquivos-scp-e-sftp)
- [Túnel SSH (port forwarding)](#tunel-ssh-port-forwarding)
- [Jump hosts / ProxyJump](#jump-hosts-proxyjump)
- [Multiplexação de conexões](#multiplexacao-de-conexoes)
- [X11 Forwarding](#x11-forwarding)
- [SSH com Git](#ssh-com-git)
- [Escape sequences](#escape-sequences)
- [Configuração do servidor (sshd_config)](#configuracao-do-servidor-sshd_config)
- [Segurança e hardening](#seguranca-e-hardening)
- [Gerenciamento de chaves](#gerenciamento-de-chaves)
- [Automação com SSH](#automacao-com-ssh)
- [SSH no Windows](#ssh-no-windows)
- [Diagnóstico e troubleshooting](#diagnostico-e-troubleshooting)

---

## <a id="conceitos-fundamentais">Conceitos fundamentais</a>

### Como funciona o SSH

O SSH opera em um modelo **cliente-servidor** na porta **22/TCP** (padrão). O fluxo de conexão é:

1. **Negociação de protocolo** — Cliente e servidor concordam na versão do protocolo (SSH-2).
2. **Troca de chaves (Key Exchange)** — Algoritmo Diffie-Hellman (ou variantes) gera uma chave de sessão compartilhada.
3. **Autenticação do servidor** — O cliente verifica a identidade do servidor via *host key* (fingerprint).
4. **Autenticação do cliente** — Por chave pública, senha ou outros métodos.
5. **Sessão criptografada** — Todo o tráfego é cifrado com a chave de sessão.

### Tipos de autenticação

| Método | Descrição | Segurança |
|---|---|---|
| **Chave pública** | Par de chaves (privada + pública) | Alta |
| **Senha** | Login com usuário e senha | Média (sujeita a brute-force) |
| **Certificado** | Chaves assinadas por uma CA SSH | Muito alta |
| **GSSAPI/Kerberos** | Autenticação integrada via tickets | Alta (ambientes corporativos) |
| **Keyboard-interactive** | Desafio-resposta (MFA, PAM) | Alta |

### Estrutura de diretórios

```text
~/.ssh/
├── authorized_keys   # Chaves públicas autorizadas (servidor)
├── config            # Configuração do cliente SSH
├── id_ed25519        # Chave privada (Ed25519)
├── id_ed25519.pub    # Chave pública (Ed25519)
├── id_rsa            # Chave privada (RSA)
├── id_rsa.pub        # Chave pública (RSA)
├── known_hosts       # Fingerprints de servidores já conectados
└── known_hosts.old   # Backup do known_hosts
```

---

## <a id="instalacao">Instalação</a>

### Linux (Debian/Ubuntu)

```bash
# Cliente SSH (geralmente já instalado)
sudo apt update && sudo apt install openssh-client

# Servidor SSH
sudo apt install openssh-server
sudo systemctl enable --now ssh
```

### Linux (RHEL/CentOS/Fedora)

```bash
sudo dnf install openssh-clients openssh-server
sudo systemctl enable --now sshd
```

### macOS

O cliente SSH já vem instalado. Para o servidor, habilite em **Preferências do Sistema → Compartilhamento → Login Remoto**.

### Windows

O OpenSSH Client já vem no Windows 10/11. Para instalar o servidor:

```powershell
# Verificar se está instalado
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'

# Instalar cliente e servidor
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# Iniciar o servidor
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
```

---

## <a id="gerar-chaves-ssh">Gerar chaves SSH</a>

### Algoritmos disponíveis

| Algoritmo | Comando | Recomendação |
|---|---|---|
| **Ed25519** | `ssh-keygen -t ed25519` | Recomendado (rápido, seguro, chaves compactas) |
| **RSA** | `ssh-keygen -t rsa -b 4096` | Compatível com sistemas legados |
| **ECDSA** | `ssh-keygen -t ecdsa -b 521` | Alternativa a RSA |
| **DSA** | `ssh-keygen -t dsa` | Obsoleto — não use |

### Gerar chave Ed25519 (recomendado)

```bash
ssh-keygen -t ed25519 -C "seu_email@example.com"
```

Saída interativa:

```text
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/user/.ssh/id_ed25519):    # Enter para aceitar
Enter passphrase (empty for no passphrase):                           # Defina uma senha forte
Enter same passphrase again:
Your identification has been saved in /home/user/.ssh/id_ed25519
Your public key has been saved in /home/user/.ssh/id_ed25519.pub
```

### Gerar chave RSA (compatibilidade)

```bash
ssh-keygen -t rsa -b 4096 -C "seu_email@example.com"
```

### Opções úteis do ssh-keygen

```bash
# Gerar chave com nome/caminho customizado
ssh-keygen -t ed25519 -f ~/.ssh/minha_chave_projeto -C "projeto-x"

# Gerar chave sem interação (sem passphrase)
ssh-keygen -t ed25519 -f ~/.ssh/chave_automacao -N "" -C "automacao"

# Alterar a passphrase de uma chave existente
ssh-keygen -p -f ~/.ssh/id_ed25519

# Ver o fingerprint de uma chave
ssh-keygen -lf ~/.ssh/id_ed25519.pub

# Ver o fingerprint em formato visual (randomart)
ssh-keygen -lvf ~/.ssh/id_ed25519.pub

# Converter chave para formato PEM
ssh-keygen -p -m PEM -f ~/.ssh/id_rsa

# Extrair a chave pública a partir da privada
ssh-keygen -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub
```

### Permissões corretas dos arquivos (obrigatório)

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519          # Chave privada
chmod 644 ~/.ssh/id_ed25519.pub      # Chave pública
chmod 600 ~/.ssh/authorized_keys     # Chaves autorizadas
chmod 600 ~/.ssh/config              # Arquivo de configuração
chmod 644 ~/.ssh/known_hosts         # Hosts conhecidos
```

---

## <a id="copiar-chave-para-servidor">Copiar chave para servidor</a>

### Método 1: ssh-copy-id (recomendado)

```bash
ssh-copy-id usuario@servidor

# Com porta alternativa
ssh-copy-id -p 2222 usuario@servidor

# Com chave específica
ssh-copy-id -i ~/.ssh/minha_chave.pub usuario@servidor
```

### Método 2: Manual via pipe

```bash
cat ~/.ssh/id_ed25519.pub | ssh usuario@servidor "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Método 3: Copiar manualmente

```bash
# No cliente — exibir a chave pública:
cat ~/.ssh/id_ed25519.pub

# No servidor — colar no authorized_keys:
echo "ssh-ed25519 AAAAC3Nza... seu_email@example.com" >> ~/.ssh/authorized_keys
```

### Verificar se a chave foi adicionada

```bash
ssh usuario@servidor "cat ~/.ssh/authorized_keys"
```

---

## <a id="conectar-ao-servidor">Conectar ao servidor</a>

### Conexão básica

```bash
ssh usuario@servidor
ssh usuario@192.168.1.100
```

### Opções comuns de conexão

```bash
# Porta alternativa
ssh -p 2222 usuario@servidor

# Chave específica
ssh -i ~/.ssh/minha_chave usuario@servidor

# IPv4 ou IPv6 forçado
ssh -4 usuario@servidor   # Forçar IPv4
ssh -6 usuario@servidor   # Forçar IPv6

# Não executar comando remoto (apenas abrir tunnel)
ssh -N usuario@servidor

# Modo em background (vai para segundo plano após autenticação)
ssh -f usuario@servidor "comando_remoto"

# Compressão (útil para conexões lentas)
ssh -C usuario@servidor

# Múltiplas opções combinadas
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null usuario@servidor
```

### Primeira conexão — verificação de host key

Na primeira conexão, o SSH pede para confirmar o fingerprint do servidor:

```text
The authenticity of host 'servidor (192.168.1.100)' can't be established.
ED25519 key fingerprint is SHA256:AbCdEf1234567890...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

Para verificar o fingerprint do servidor antes de conectar:

```bash
# No servidor, executar:
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

---

## <a id="executar-comandos-remotos">Executar comandos remotos</a>

### Comando único

```bash
ssh usuario@servidor "uptime"
ssh usuario@servidor "df -h"
ssh usuario@servidor "cat /etc/os-release"
```

### Múltiplos comandos

```bash
ssh usuario@servidor "cd /var/log && tail -n 50 syslog && df -h"
```

### Executar script local no servidor remoto

```bash
ssh usuario@servidor 'bash -s' < script_local.sh

# Com argumentos
ssh usuario@servidor 'bash -s' < script_local.sh arg1 arg2
```

### Executar com sudo

```bash
# Interativo (pede senha)
ssh -t usuario@servidor "sudo systemctl restart nginx"

# O -t força a alocação de pseudo-terminal (necessário para sudo)
```

### Heredoc remoto

```bash
ssh usuario@servidor << 'EOF'
echo "Hostname: $(hostname)"
echo "Data: $(date)"
echo "Uptime: $(uptime)"
EOF
```

### Executar em múltiplos servidores

```bash
for host in servidor1 servidor2 servidor3; do
    echo "=== $host ==="
    ssh usuario@$host "uptime && df -h /"
done
```

---

## <a id="arquivo-de-configuracao-do-cliente">Arquivo de configuração do cliente</a>

O arquivo `~/.ssh/config` permite definir configurações por host, evitando repetir parâmetros.

### Exemplo básico

```text
Host meu-servidor
    HostName servidor.exemplo.com
    Port 2222
    User usuario
    IdentityFile ~/.ssh/id_ed25519
```

Então basta `ssh meu-servidor`.

### Exemplo completo com múltiplos hosts

```text
# Configurações globais (aplicam-se a todos os hosts)
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
    IdentitiesOnly yes
    HashKnownHosts yes

# Servidor de produção
Host producao
    HostName prod.exemplo.com
    Port 22
    User deploy
    IdentityFile ~/.ssh/id_ed25519_prod
    ForwardAgent no

# Servidor de staging
Host staging
    HostName staging.exemplo.com
    User deploy
    IdentityFile ~/.ssh/id_ed25519_staging
    LocalForward 5432 localhost:5432

# Servidor de desenvolvimento via bastion
Host dev
    HostName 10.0.1.50
    User dev
    ProxyJump bastion
    IdentityFile ~/.ssh/id_ed25519_dev

# Bastion host (jump box)
Host bastion
    HostName bastion.exemplo.com
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_bastion
    ForwardAgent yes

# Padrão com wildcard — todos os hosts internos
Host 10.0.*
    User admin
    ProxyJump bastion
    IdentityFile ~/.ssh/id_ed25519_internal
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

# GitHub
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github

# GitLab
Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_ed25519_gitlab
    PreferredAuthentications publickey
```

### Diretivas mais úteis

| Diretiva | Descrição |
|---|---|
| `HostName` | Endereço real do host (IP ou domínio) |
| `Port` | Porta do SSH (padrão: 22) |
| `User` | Usuário para login |
| `IdentityFile` | Caminho da chave privada |
| `IdentitiesOnly` | Usar apenas a chave especificada (yes/no) |
| `ProxyJump` | Host intermediário (jump/bastion) |
| `ProxyCommand` | Comando para conectar ao host (mais flexível que ProxyJump) |
| `ForwardAgent` | Encaminhar o agente SSH (yes/no) |
| `LocalForward` | Port forwarding local |
| `RemoteForward` | Port forwarding remoto |
| `DynamicForward` | SOCKS proxy dinâmico |
| `ServerAliveInterval` | Intervalo de keep-alive (segundos) |
| `ServerAliveCountMax` | Máximo de keep-alives sem resposta |
| `StrictHostKeyChecking` | Verificar host key (yes/ask/no) |
| `UserKnownHostsFile` | Arquivo de hosts conhecidos |
| `Compression` | Habilitar compressão (yes/no) |
| `LogLevel` | Nível de log (QUIET, FATAL, ERROR, INFO, VERBOSE, DEBUG) |
| `AddKeysToAgent` | Adicionar chave automaticamente ao agente (yes/no) |
| `ControlMaster` | Multiplexação de conexões |
| `ControlPath` | Caminho do socket de controle |
| `ControlPersist` | Tempo para manter conexão master ativa |
| `RequestTTY` | Forçar/desabilitar pseudo-terminal (yes/no/force/auto) |
| `SendEnv` | Variáveis de ambiente para enviar ao servidor |

### Ordem de precedência

O SSH lê as configurações nesta ordem (a primeira encontrada prevalece):

1. Opções de linha de comando (`-o`, `-p`, `-i`, etc.)
2. `~/.ssh/config` (configuração do usuário)
3. `/etc/ssh/ssh_config` (configuração global do sistema)

---

## <a id="agente-ssh">Agente SSH</a>

O `ssh-agent` armazena chaves privadas descriptografadas na memória, evitando digitar a passphrase a cada uso.

### Iniciar o agente

```bash
# Linux / macOS
eval "$(ssh-agent -s)"

# Verificar se está rodando
echo $SSH_AUTH_SOCK
```

### Adicionar chaves

```bash
# Adicionar a chave padrão
ssh-add ~/.ssh/id_ed25519

# Adicionar chave específica
ssh-add ~/.ssh/minha_chave

# Adicionar todas as chaves padrão (~/.ssh/id_*)
ssh-add

# Adicionar com tempo de vida (expira em 1 hora)
ssh-add -t 3600 ~/.ssh/id_ed25519
```

### Listar e remover chaves

```bash
# Listar chaves carregadas
ssh-add -l

# Listar chaves com fingerprint completo
ssh-add -L

# Remover uma chave específica
ssh-add -d ~/.ssh/id_ed25519

# Remover todas as chaves
ssh-add -D
```

### Encaminhamento de agente (Agent Forwarding)

Permite que servidores intermediários usem suas chaves locais, sem copiar a chave privada para o servidor.

```bash
# Via linha de comando
ssh -A usuario@bastion

# Depois, no bastion, o agente estará disponível:
ssh usuario@servidor-interno   # Usa a chave local transparentemente
```

No `~/.ssh/config`:

```text
Host bastion
    ForwardAgent yes
```

**Atenção:** habilite Agent Forwarding apenas para hosts confiáveis, pois um administrador do servidor intermediário pode usar seu agente enquanto a conexão estiver ativa.

### Agente SSH no macOS (Keychain)

```bash
# Adicionar chave ao Keychain (persistente após reboot)
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

No `~/.ssh/config`:

```text
Host *
    UseKeychain yes
    AddKeysToAgent yes
```

### Agente SSH no Linux (persistente com systemd)

Crie o serviço `~/.config/systemd/user/ssh-agent.service`:

```ini
[Unit]
Description=SSH authentication agent

[Service]
Type=simple
Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
ExecStart=/usr/bin/ssh-agent -D -a $SSH_AUTH_SOCK
ExecStartPost=/usr/bin/ssh-add

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now ssh-agent
echo 'export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"' >> ~/.bashrc
```

---

## <a id="transferencia-de-arquivos-scp-e-sftp">Transferência de arquivos (SCP e SFTP)</a>

### SCP (Secure Copy)

```bash
# Copiar arquivo local para servidor
scp arquivo.txt usuario@servidor:/caminho/destino/

# Copiar arquivo do servidor para local
scp usuario@servidor:/caminho/arquivo.txt ./

# Copiar diretório recursivamente
scp -r pasta/ usuario@servidor:/caminho/destino/

# Com porta alternativa
scp -P 2222 arquivo.txt usuario@servidor:/tmp/

# Com chave específica
scp -i ~/.ssh/minha_chave arquivo.txt usuario@servidor:/tmp/

# Preservar timestamps e permissões
scp -rp pasta/ usuario@servidor:/caminho/destino/

# Limitar largura de banda (em Kbit/s)
scp -l 1000 arquivo_grande.tar.gz usuario@servidor:/tmp/

# Copiar entre dois servidores remotos (passando pelo cliente)
scp usuario@servidor1:/arquivo.txt usuario@servidor2:/destino/
```

### SFTP (SSH File Transfer Protocol)

SFTP oferece uma sessão interativa mais completa que o SCP.

```bash
# Conectar
sftp usuario@servidor

# Conectar em porta alternativa
sftp -P 2222 usuario@servidor
```

Comandos dentro da sessão SFTP:

```text
# Navegação
ls                          # Listar arquivos remotos
lls                         # Listar arquivos locais
cd /var/log                 # Mudar diretório remoto
lcd ~/downloads             # Mudar diretório local
pwd                         # Diretório remoto atual
lpwd                        # Diretório local atual

# Transferência
get arquivo.txt             # Baixar arquivo
get -r pasta/               # Baixar diretório
put arquivo.txt             # Enviar arquivo
put -r pasta/               # Enviar diretório

# Operações
mkdir nova_pasta            # Criar diretório remoto
rm arquivo.txt              # Remover arquivo remoto
rename antigo.txt novo.txt  # Renomear arquivo remoto
chmod 644 arquivo.txt       # Alterar permissões

# Sair
exit
```

### rsync via SSH (recomendado para sincronização)

```bash
# Sincronizar diretório local para remoto
rsync -avz --progress pasta/ usuario@servidor:/caminho/destino/

# Sincronizar do remoto para local
rsync -avz usuario@servidor:/caminho/origem/ pasta_local/

# Com porta alternativa
rsync -avz -e "ssh -p 2222" pasta/ usuario@servidor:/destino/

# Excluir arquivos
rsync -avz --exclude='*.log' --exclude='node_modules/' pasta/ usuario@servidor:/destino/

# Dry-run (simular sem executar)
rsync -avzn pasta/ usuario@servidor:/destino/

# Deletar arquivos no destino que não existem na origem
rsync -avz --delete pasta/ usuario@servidor:/destino/
```

---

## <a id="tunel-ssh-port-forwarding">Túnel SSH (port forwarding)</a>

### Local Port Forwarding (-L)

Expõe uma porta do servidor remoto na máquina local.

```bash
# Acessar banco de dados remoto (porta 5432) via localhost:5432
ssh -L 5432:localhost:5432 usuario@servidor

# Acessar aplicação web remota na porta 8080 localmente na 3000
ssh -L 3000:localhost:8080 usuario@servidor

# Acessar um host interno (db-server) através do bastion
ssh -L 5432:db-server:5432 usuario@bastion

# Bind em todas as interfaces (permite acesso por outros hosts na rede)
ssh -L 0.0.0.0:5432:db-server:5432 usuario@bastion

# Combinação com -N (sem terminal) e -f (background)
ssh -fNL 5432:localhost:5432 usuario@servidor
```

Diagrama do fluxo:

```text
[Máquina Local:5432] --SSH--> [Servidor] ---> [db-server:5432]
         ↑                                          ↑
     você acessa                              destino real
```

### Remote Port Forwarding (-R)

Expõe uma porta da máquina local no servidor remoto.

```bash
# Servidor remoto escuta na porta 8080 e redireciona para sua máquina local na 3000
ssh -R 8080:localhost:3000 usuario@servidor

# Combinação com -N e -f
ssh -fNR 8080:localhost:3000 usuario@servidor
```

Diagrama do fluxo:

```text
[Servidor Remoto:8080] --SSH--> [Máquina Local:3000]
         ↑                              ↑
 outros acessam aqui            sua aplicação roda aqui
```

Para habilitar Remote Forwarding no servidor, adicione ao `sshd_config`:

```text
GatewayPorts yes
```

### Dynamic Port Forwarding (-D) — SOCKS Proxy

Cria um proxy SOCKS na máquina local, roteando todo o tráfego pelo servidor remoto.

```bash
# Criar proxy SOCKS5 na porta 1080
ssh -D 1080 usuario@servidor

# Em background
ssh -fND 1080 usuario@servidor
```

Use no navegador ou em aplicações configurando proxy SOCKS5 em `localhost:1080`.

Uso com `curl`:

```bash
curl --socks5-hostname localhost:1080 http://site-interno.exemplo.com
```

### Manter túneis persistentes com autossh

`autossh` reconecta automaticamente túneis que caem.

```bash
# Instalar
sudo apt install autossh

# Túnel local persistente
autossh -M 0 -fNL 5432:localhost:5432 usuario@servidor \
    -o "ServerAliveInterval=30" \
    -o "ServerAliveCountMax=3"

# Túnel reverso persistente
autossh -M 0 -fNR 8080:localhost:3000 usuario@servidor
```

O `-M 0` desabilita a porta de monitoramento do autossh, usando o keepalive do SSH.

---

## <a id="jump-hosts-proxyjump">Jump hosts / ProxyJump</a>

Conectar a servidores internos através de um bastion host (jump box).

### ProxyJump (SSH 7.3+, recomendado)

```bash
# Via linha de comando
ssh -J usuario@bastion usuario@servidor-interno

# Múltiplos saltos
ssh -J usuario@bastion1,usuario@bastion2 usuario@destino

# Com porta alternativa no bastion
ssh -J usuario@bastion:2222 usuario@servidor-interno
```

### ProxyJump no config

```text
Host bastion
    HostName bastion.exemplo.com
    User admin

Host servidor-interno
    HostName 10.0.1.50
    User deploy
    ProxyJump bastion
```

### ProxyCommand (alternativa para versões antigas)

```text
Host servidor-interno
    HostName 10.0.1.50
    User deploy
    ProxyCommand ssh -W %h:%p bastion
```

### ProxyCommand com netcat (ambientes restritos)

```text
Host servidor-interno
    ProxyCommand ssh bastion nc %h %p
```

---

## <a id="multiplexacao-de-conexoes">Multiplexação de conexões</a>

Reutiliza uma única conexão TCP para múltiplas sessões SSH ao mesmo host, eliminando o overhead de reconexão.

### Configuração

No `~/.ssh/config`:

```text
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600
```

Crie o diretório de sockets:

```bash
mkdir -p ~/.ssh/sockets
```

### Como funciona

- **ControlMaster auto** — A primeira conexão cria o socket master. Conexões subsequentes reutilizam.
- **ControlPath** — Caminho do socket Unix. `%r` = usuário, `%h` = host, `%p` = porta.
- **ControlPersist 600** — Mantém a conexão master por 600 segundos após a última sessão fechar.

### Gerenciar conexões multiplexadas

```bash
# Verificar status da conexão master
ssh -O check usuario@servidor

# Encerrar a conexão master
ssh -O exit usuario@servidor

# Forçar encerramento
ssh -O stop usuario@servidor
```

---

## <a id="x11-forwarding">X11 Forwarding</a>

Permite executar aplicações gráficas remotas exibindo-as localmente.

```bash
# Habilitar X11 Forwarding
ssh -X usuario@servidor

# Modo trusted (menos seguro, mais compatível)
ssh -Y usuario@servidor
```

No servidor, o `sshd_config` precisa ter:

```text
X11Forwarding yes
```

Exemplo de uso:

```bash
ssh -X usuario@servidor
firefox &           # Abre o Firefox remoto, exibindo localmente
xeyes &             # Teste rápido de X11
```

---

## <a id="ssh-com-git">SSH com Git</a>

### Configurar chave SSH para GitHub/GitLab

```bash
# Gerar chave dedicada
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github -C "seu_email@example.com"

# Copiar a chave pública
cat ~/.ssh/id_ed25519_github.pub
# Cole em: GitHub → Settings → SSH and GPG keys → New SSH key
```

### Testar a conexão

```bash
ssh -T git@github.com
# Hi usuario! You've successfully authenticated...

ssh -T git@gitlab.com
```

### Configuração para múltiplas contas Git

```text
# Conta pessoal
Host github-pessoal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_pessoal

# Conta do trabalho
Host github-trabalho
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_trabalho
```

Clonar repositórios usando o alias:

```bash
git clone git@github-pessoal:usuario/repo.git
git clone git@github-trabalho:empresa/repo.git
```

---

## <a id="escape-sequences">Escape sequences</a>

Durante uma sessão SSH, pressione `~` (til) no início de uma nova linha seguido de um caractere de controle:

| Sequência | Ação |
|---|---|
| `~.` | Encerrar conexão (útil quando a sessão trava) |
| `~^Z` | Suspender a sessão SSH (voltar com `fg`) |
| `~#` | Listar conexões encaminhadas |
| `~&` | Colocar SSH em background ao aguardar encerramento |
| `~?` | Mostrar lista de escape sequences |
| `~C` | Abrir linha de comando SSH (para adicionar tunnels dinamicamente) |
| `~R` | Solicitar rekeying da conexão |

### Adicionar túnel dinamicamente

Dentro de uma sessão ativa, pressione `~C` e depois:

```text
ssh> -L 5432:localhost:5432    # Adicionar local forward
ssh> -R 8080:localhost:3000    # Adicionar remote forward
ssh> -D 1080                   # Adicionar dynamic forward
ssh> -KL 5432                  # Remover local forward
```

---

## <a id="configuracao-do-servidor-sshd_config">Configuração do servidor (sshd_config)</a>

O arquivo `/etc/ssh/sshd_config` controla o comportamento do daemon SSH.

### Configuração recomendada para segurança

```text
# /etc/ssh/sshd_config

# Protocolo e rede
Port 2222                                     # Porta não-padrão
AddressFamily inet                            # IPv4 apenas (ou inet6, any)
ListenAddress 0.0.0.0                         # Interface de escuta

# Autenticação
PermitRootLogin no                            # Nunca permitir login como root
PasswordAuthentication no                     # Desabilitar login por senha
PubkeyAuthentication yes                      # Habilitar autenticação por chave
AuthorizedKeysFile .ssh/authorized_keys       # Arquivo de chaves autorizadas
PermitEmptyPasswords no                       # Nunca permitir senhas vazias
MaxAuthTries 3                                # Máximo de tentativas de autenticação
MaxSessions 5                                 # Máximo de sessões por conexão
LoginGraceTime 30                             # Tempo (segundos) para autenticar
AuthenticationMethods publickey               # Métodos permitidos

# Segurança
X11Forwarding no                              # Desabilitar X11 (se não usar)
AllowTcpForwarding yes                        # Permitir tunneling
AllowAgentForwarding no                       # Desabilitar agent forwarding
PermitTunnel no                               # Desabilitar tunnel de camada 2/3
GatewayPorts no                               # Não permitir bind em interfaces externas

# Usuários e grupos permitidos
AllowUsers deploy admin                       # Apenas esses usuários podem conectar
# AllowGroups ssh-users                       # Ou permitir por grupo
DenyUsers root test                           # Negar explicitamente estes usuários

# Banner e informações
Banner /etc/ssh/banner.txt                    # Mensagem antes do login
PrintMotd yes                                 # Mostrar /etc/motd após login
PrintLastLog yes                              # Mostrar último login

# Keep-alive
ClientAliveInterval 300                       # Ping a cada 5 minutos
ClientAliveCountMax 2                         # Desconectar após 2 falhas

# Criptografia forte
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256

# Logging
SyslogFacility AUTH
LogLevel VERBOSE

# SFTP
Subsystem sftp /usr/lib/openssh/sftp-server
```

### Após alterar o sshd_config

```bash
# Validar a configuração antes de reiniciar
sudo sshd -t

# Reiniciar o serviço
sudo systemctl restart sshd

# Acompanhar logs
sudo journalctl -u sshd -f
```

**Importante:** mantenha uma sessão SSH aberta enquanto testa alterações — se errar a configuração, não será desconectado da sessão existente.

### Autenticação multifator (MFA)

Combinar chave pública + código TOTP:

```text
# No sshd_config
AuthenticationMethods publickey,keyboard-interactive
ChallengeResponseAuthentication yes
```

Instalar e configurar `libpam-google-authenticator`:

```bash
sudo apt install libpam-google-authenticator
google-authenticator   # Gerar código QR para o app autenticador
```

Adicionar ao `/etc/pam.d/sshd`:

```text
auth required pam_google_authenticator.so
```

### Restringir comandos por chave

No `authorized_keys`, prefixe a chave com opções:

```text
command="/usr/bin/backup.sh",no-port-forwarding,no-X11-forwarding,no-agent-forwarding ssh-ed25519 AAAAC3Nza... backup@server
```

A chave só pode executar o comando especificado.

### Chroot para SFTP

Restringir usuário SFTP para um diretório específico:

```text
# No sshd_config
Match User sftpuser
    ChrootDirectory /home/sftpuser
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

Permissões necessárias:

```bash
sudo chown root:root /home/sftpuser
sudo chmod 755 /home/sftpuser
sudo mkdir /home/sftpuser/uploads
sudo chown sftpuser:sftpuser /home/sftpuser/uploads
```

---

## <a id="seguranca-e-hardening">Segurança e hardening</a>

### Checklist de segurança

- [ ] Desativar login por senha (`PasswordAuthentication no`)
- [ ] Desativar login do root (`PermitRootLogin no`)
- [ ] Usar chaves Ed25519 com passphrase
- [ ] Usar porta não-padrão (ex: 2222)
- [ ] Limitar usuários permitidos (`AllowUsers` ou `AllowGroups`)
- [ ] Configurar Fail2Ban ou similar
- [ ] Manter o OpenSSH atualizado
- [ ] Configurar algoritmos de criptografia fortes
- [ ] Habilitar logging (`LogLevel VERBOSE`)
- [ ] Implementar MFA quando possível
- [ ] Limitar tempo de autenticação (`LoginGraceTime`)
- [ ] Configurar keep-alive adequado
- [ ] Revogar chaves antigas/não utilizadas regularmente

### Fail2Ban para SSH

```bash
sudo apt install fail2ban
```

Criar `/etc/fail2ban/jail.local`:

```ini
[sshd]
enabled  = true
port     = 2222
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 3600
findtime = 600
action   = iptables-multiport[name=sshd, port="2222", protocol=tcp]
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd        # Ver status
sudo fail2ban-client set sshd unbanip 1.2.3.4  # Desbanir IP
```

### Firewall (UFW)

```bash
# Permitir SSH na porta 2222
sudo ufw allow 2222/tcp

# Limitar taxa de conexão (max 6 conexões em 30 segundos)
sudo ufw limit 2222/tcp

sudo ufw enable
```

### Firewall (iptables)

```bash
# Permitir SSH na porta 2222
sudo iptables -A INPUT -p tcp --dport 2222 -j ACCEPT

# Rate limiting (max 3 novas conexões por minuto)
sudo iptables -A INPUT -p tcp --dport 2222 -m state --state NEW -m recent --set
sudo iptables -A INPUT -p tcp --dport 2222 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
```

### Port knocking

Técnica que exige uma sequência de conexões em portas específicas antes de abrir a porta SSH.

```bash
# Instalar
sudo apt install knockd

# Configurar /etc/knockd.conf
```

```ini
[options]
    UseSyslog

[openSSH]
    sequence    = 7000,8000,9000
    seq_timeout = 10
    command     = /sbin/iptables -A INPUT -s %IP% -p tcp --dport 2222 -j ACCEPT
    tcpflags    = syn

[closeSSH]
    sequence    = 9000,8000,7000
    seq_timeout = 10
    command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 2222 -j ACCEPT
    tcpflags    = syn
```

No cliente:

```bash
# Usando knock
knock servidor 7000 8000 9000
ssh usuario@servidor -p 2222

# Usando nmap como alternativa
for port in 7000 8000 9000; do nmap -Pn --host-timeout 100 -p $port servidor; done
```

### Auditar configuração SSH

```bash
# Verificar versão do OpenSSH
ssh -V

# Auditar com ssh-audit (ferramenta de terceiros)
pip install ssh-audit
ssh-audit servidor.exemplo.com

# Testar algoritmos aceitos pelo servidor
nmap --script ssh2-enum-algos -sV -p 22 servidor.exemplo.com
```

---

## <a id="gerenciamento-de-chaves">Gerenciamento de chaves</a>

### Rotação de chaves

```bash
# 1. Gerar nova chave
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_new -C "nova-chave-$(date +%Y%m)"

# 2. Adicionar nova chave ao servidor
ssh-copy-id -i ~/.ssh/id_ed25519_new.pub usuario@servidor

# 3. Testar a nova chave
ssh -i ~/.ssh/id_ed25519_new usuario@servidor

# 4. Remover a chave antiga do authorized_keys no servidor
ssh usuario@servidor "sed -i '/chave_antiga_fingerprint/d' ~/.ssh/authorized_keys"

# 5. Atualizar ~/.ssh/config com a nova chave
```

### Revogar uma chave

No servidor, remova a linha correspondente do `~/.ssh/authorized_keys`, ou use `RevokedKeys`:

```text
# No sshd_config
RevokedKeys /etc/ssh/revoked_keys
```

```bash
# Adicionar chave revogada
sudo ssh-keygen -kf /etc/ssh/revoked_keys ~/.ssh/chave_comprometida.pub
```

### Certificados SSH (escala)

Uma alternativa ao `authorized_keys` para ambientes grandes, onde uma CA assina chaves de usuários e hosts.

```bash
# 1. Criar a CA
ssh-keygen -t ed25519 -f /etc/ssh/ca_key -C "SSH CA"

# 2. Assinar chave de usuário (válida por 52 semanas)
ssh-keygen -s /etc/ssh/ca_key -I "usuario@empresa" -n usuario -V +52w ~/.ssh/id_ed25519.pub
# Gera: ~/.ssh/id_ed25519-cert.pub

# 3. Assinar chave de host
ssh-keygen -s /etc/ssh/ca_key -I "servidor.exemplo.com" -h -n servidor.exemplo.com -V +52w /etc/ssh/ssh_host_ed25519_key.pub

# 4. No servidor (sshd_config) — confiar na CA para usuários
TrustedUserCAKeys /etc/ssh/ca_key.pub

# 5. No cliente (ssh_config ou known_hosts) — confiar na CA para hosts
@cert-authority *.exemplo.com ssh-ed25519 AAAAC3Nza... SSH CA
```

### known_hosts seguro

```bash
# Remover um host específico
ssh-keygen -R servidor.exemplo.com

# Hashing do known_hosts (esconde nomes de host)
ssh-keygen -H -f ~/.ssh/known_hosts
```

---

## <a id="automacao-com-ssh">Automação com SSH</a>

### Opções para scripts

```bash
# Desabilitar verificação de host (use com cautela, apenas em ambientes controlados)
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null usuario@servidor "comando"

# Forçar autenticação por chave (falha se a chave não funcionar)
ssh -o BatchMode=yes -o PasswordAuthentication=no usuario@servidor "comando"

# Timeout de conexão
ssh -o ConnectTimeout=10 usuario@servidor "comando"
```

### sshpass (login com senha em scripts)

```bash
sudo apt install sshpass

# Via variável de ambiente (mais seguro que argumento)
export SSHPASS="minha_senha"
sshpass -e ssh usuario@servidor "comando"

# Via arquivo
sshpass -f /caminho/senha.txt ssh usuario@servidor "comando"
```

**Nota:** Use `sshpass` apenas em situações onde chaves não são viáveis. Sempre prefira chaves SSH.

### Ansible e SSH

O Ansible usa SSH nativamente. Configure em `ansible.cfg`:

```ini
[ssh_connection]
ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
```

### Expect (automação interativa)

```bash
#!/usr/bin/expect -f
spawn ssh usuario@servidor
expect "password:"
send "minha_senha\r"
expect "$ "
send "uptime\r"
expect "$ "
send "exit\r"
```

### Script de deploy via SSH

```bash
#!/bin/bash
set -euo pipefail

SERVER="deploy@producao"
APP_DIR="/opt/app"

echo "==> Fazendo deploy..."

# Upload do pacote
scp -q dist/app.tar.gz "$SERVER:$APP_DIR/"

# Executar deploy remoto
ssh "$SERVER" << 'DEPLOY'
    set -euo pipefail
    cd /opt/app
    tar xzf app.tar.gz
    docker compose down
    docker compose up -d
    rm app.tar.gz
    echo "Deploy concluído em $(date)"
DEPLOY

echo "==> Deploy finalizado com sucesso!"
```

---

## <a id="ssh-no-windows">SSH no Windows</a>

### OpenSSH nativo (Windows 10/11)

```powershell
# Gerar chave
ssh-keygen -t ed25519 -C "seu_email@example.com"

# Chaves ficam em C:\Users\SeuUsuario\.ssh\

# Conectar
ssh usuario@servidor
```

### Configurar o ssh-agent no Windows

```powershell
# Habilitar e iniciar o serviço
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent

# Adicionar chave
ssh-add $env:USERPROFILE\.ssh\id_ed25519

# Listar chaves
ssh-add -l
```

### PuTTY e PuTTYgen

- **PuTTYgen** — Gerar/converter chaves (formato `.ppk`).
- **Pageant** — Agente SSH do PuTTY.
- **PuTTY** — Cliente SSH com interface gráfica.

Converter chave OpenSSH para PuTTY:

```text
PuTTYgen → Conversions → Import key → Selecionar id_ed25519 → Save private key (.ppk)
```

### WSL (Windows Subsystem for Linux)

No WSL, o SSH funciona como em qualquer distribuição Linux. Para compartilhar chaves entre Windows e WSL:

```bash
# Copiar chaves do Windows para WSL
cp -r /mnt/c/Users/SeuUsuario/.ssh/ ~/.ssh/
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
```

---

## <a id="diagnostico-e-troubleshooting">Diagnóstico e troubleshooting</a>

### Níveis de verbose

```bash
ssh -v usuario@servidor    # Verbose (nível 1)
ssh -vv usuario@servidor   # Mais detalhes (nível 2)
ssh -vvv usuario@servidor  # Máximo de detalhes (nível 3)
```

### Problemas comuns e soluções

| Problema | Causa provável | Solução |
|---|---|---|
| `Permission denied (publickey)` | Chave não reconhecida | Verificar `authorized_keys`, permissões, `IdentityFile` |
| `Connection refused` | SSH não rodando ou porta errada | `sudo systemctl status sshd`, verificar porta |
| `Connection timed out` | Firewall bloqueando | Verificar firewall, porta, IP |
| `Host key verification failed` | Host key mudou | `ssh-keygen -R servidor` e reconectar |
| `Too many authentication failures` | Muitas chaves sendo testadas | Usar `IdentitiesOnly yes` no config |
| `broken pipe` | Conexão ociosa desconectada | Configurar `ServerAliveInterval` |
| `Could not open a connection to your authentication agent` | ssh-agent não rodando | `eval "$(ssh-agent -s)"` |
| `Bad owner or permissions` | Permissões incorretas em ~/.ssh | `chmod 700 ~/.ssh; chmod 600 ~/.ssh/*` |

### Verificações no servidor

```bash
# Status do serviço SSH
sudo systemctl status sshd

# Ver logs em tempo real
sudo journalctl -u sshd -f

# Verificar portas abertas
sudo ss -tlnp | grep ssh

# Testar configuração do sshd
sudo sshd -t

# Testar com arquivo de configuração específico
sudo sshd -t -f /etc/ssh/sshd_config

# Verificar permissões
ls -la ~/.ssh/
ls -la /etc/ssh/

# Ver conexões ativas
who
w
ss -tnp | grep :22
```

### Verificações no cliente

```bash
# Testar conectividade básica
ping servidor
telnet servidor 22
nc -zv servidor 22

# Verificar chaves carregadas no agente
ssh-add -l

# Verificar fingerprint do servidor
ssh-keyscan -t ed25519 servidor

# Testar com chave específica e verbose
ssh -vvv -i ~/.ssh/minha_chave usuario@servidor
```

### Debug do lado do servidor

Rodar o sshd em modo debug temporário (em outra porta):

```bash
sudo /usr/sbin/sshd -d -p 2222

# No cliente, conectar na porta de debug:
ssh -p 2222 usuario@servidor
```

O output detalhado no servidor mostra cada etapa da autenticação.

---

Referências:

- `man ssh` — Manual do cliente SSH
- `man sshd` — Manual do daemon SSH  
- `man ssh_config` — Configuração do cliente
- `man sshd_config` — Configuração do servidor
- `man ssh-keygen` — Geração e gerenciamento de chaves
- `man ssh-agent` — Agente de autenticação
- `man ssh-copy-id` — Instalação de chaves
- `man scp` / `man sftp` — Transferência de arquivos
- [OpenSSH Manual Pages](https://www.openssh.com/manual.html)
- [SSH Academy](https://www.ssh.com/academy/ssh)
