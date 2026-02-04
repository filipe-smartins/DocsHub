# SSH - Guia Rápido

SSH (Secure Shell) permite acesso remoto seguro a servidores. Este guia cobre os comandos e práticas mais usados.

## Índice

- [Gerar chaves SSH](#gerar-chaves-ssh)
- [Copiar chave para servidor](#copiar-chave-para-servidor)
- [Conectar ao servidor](#conectar-ao-servidor)
- [Arquivo de configuração](#arquivo-de-configuracao)
- [Agente SSH](#agente-ssh)
- [Túnel SSH (port forwarding)](#tunel-ssh-port-forwarding)
- [Segurança e boas práticas](#seguranca-e-boas-praticas)
- [Diagnóstico](#diagnostico)

## <a id="gerar-chaves-ssh">Gerar chaves SSH</a>

```bash
ssh-keygen -t ed25519 -C "seu_email@example.com"
```

Chaves ficam em `~/.ssh/id_ed25519` (privada) e `~/.ssh/id_ed25519.pub` (pública).

## <a id="copiar-chave-para-servidor">Copiar chave para servidor</a>

```bash
ssh-copy-id usuario@servidor
# ou manualmente:
cat ~/.ssh/id_ed25519.pub | ssh usuario@servidor "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

## <a id="conectar-ao-servidor">Conectar ao servidor</a>

```bash
ssh usuario@servidor
ssh -p 2222 usuario@servidor  # porta alternativa
```

## <a id="arquivo-de-configuracao">Arquivo de configuração</a>

Em `~/.ssh/config` você pode simplificar acessos:

```text
Host meu-servidor
	HostName servidor.exemplo.com
	Port 2222
	User usuario
	IdentityFile ~/.ssh/id_ed25519
```

Então basta `ssh meu-servidor`.

## <a id="agente-ssh">Agente SSH</a>

Use `ssh-agent` para carregar chaves e evitar digitar a senha sempre:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

## <a id="tunel-ssh-port-forwarding">Túnel SSH (port forwarding)</a>

Criar túnel local:

```bash
ssh -L 5432:localhost:5432 usuario@servidor  # expõe porta remota localmente
```

## <a id="seguranca-e-boas-praticas">Segurança e boas práticas</a>
- Proteja a chave privada com senha.
- Desative login por senha no servidor: `PasswordAuthentication no` em `/etc/ssh/sshd_config`.
- Use `Fail2Ban` ou firewall para limitar tentativas.

## <a id="diagnostico">Diagnóstico</a>

- `ssh -v usuario@servidor` para debug.
- Verifique permissões do diretório `~/.ssh` (700) e `authorized_keys` (600).

---
Referências: `man ssh`, `man sshd`, `man ssh_config`.
