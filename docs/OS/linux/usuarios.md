# Administração Linux — Usuários

Comandos e práticas para gerenciar usuários e grupos.

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Adicionar usuário com home e shell:

```bash
sudo useradd -m -s /bin/bash novo_usuario
sudo passwd novo_usuario
```

Adicionar e remover com ferramentas mais amigáveis:

```bash
sudo adduser novo_usuario   # Debian/Ubuntu (interativo)
sudo deluser usuario        # remove usuário (use --remove-home para apagar home)
```

Grupos:

```bash
sudo groupadd devs
sudo usermod -aG devs usuario  # adicionar usuário ao grupo
groups usuario                  # listar grupos do usuário
```

Permissões de sudo:

```bash
sudo visudo                   # editar /etc/sudoers com segurança
sudo usermod -aG sudo usuario # dar acesso sudo (Debian/Ubuntu)
```

Bloquear/desbloquear conta:

```bash
sudo passwd -l usuario   # bloquear
sudo passwd -u usuario   # desbloquear
```

Chaves SSH:

```bash
sudo -u usuario mkdir -p /home/usuario/.ssh
sudo -u usuario chmod 700 /home/usuario/.ssh
sudo -u usuario touch /home/usuario/.ssh/authorized_keys
sudo -u usuario chmod 600 /home/usuario/.ssh/authorized_keys
```

Boas práticas:
- Use `adduser` quando disponível (interativo e seguro).
- Prefira `groups`/roles em vez de dar privilégios individuais.
- Revise contas inativas e remova ou bloqueie quando apropriado.
