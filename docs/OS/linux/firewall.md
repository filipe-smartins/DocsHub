# Redes & Segurança — Firewall (ufw / iptables)

Este guia apresenta comandos básicos para `ufw` (fácil) e `iptables` (baixo nível).

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

UFW (Uncomplicated Firewall) — rápido e recomendado para servidores simples:

```bash
sudo ufw status verbose
sudo ufw allow 22/tcp        # permitir SSH
sudo ufw allow 80/tcp        # permitir HTTP
sudo ufw allow 443/tcp       # permitir HTTPS
sudo ufw limit 22/tcp        # proteger contra brute-force
sudo ufw deny 23/tcp         # negar Telnet
sudo ufw enable
sudo ufw disable
sudo ufw delete allow 80/tcp
```

Iptables — exemplo básico para permitir SSH e tráfego estabelecido:

```bash
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -p icmp -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
```

Salvar e restaurar regras:

```bash
sudo iptables-save > /etc/iptables.rules
sudo iptables-restore < /etc/iptables.rules
```

Nota sobre `nftables`:
- `nftables` é a substituta moderna do `iptables` em muitas distros.

Boas práticas:
- Teste regras em uma sessão remota separada antes de aplicar (evite se trancar fora via SSH).
- Automatize restauração de regras no boot (systemd unit ou scripts de init).
- Combine firewall com `fail2ban` para bloquear IPs maliciosos.
