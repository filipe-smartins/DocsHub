# Rede - Conceitos e comandos básicos (Linux)

Comandos úteis para diagnosticar e configurar rede em Linux.

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Ver interfaces e endereços:

```bash
ip addr show
ip -br addr  # saída compacta
```

Ativar/desativar interface:

```bash
sudo ip link set dev eth0 up
sudo ip link set dev eth0 down
```

Testes de conectividade:

```bash
ping -c 4 8.8.8.8
traceroute example.com   # ou `traceroute -n`
```

Ver sockets e portas:

```bash
ss -tuln
netstat -tuln  # alternativa em sistemas mais antigos
```

Rotas e gateway:

```bash
ip route show
sudo ip route add default via 192.168.1.1
```

DNS:

```bash
cat /etc/resolv.conf
dig +short example.com @8.8.8.8
```

Recarregar serviços de rede (systemd):

```bash
sudo systemctl restart NetworkManager
sudo systemctl restart networking  # distribuições Debian-like
```

Dica: use `tcpdump` para capturar tráfego e `nmcli` para gerenciar conexões NetworkManager.
