# Administração Linux — systemd

Conceitos e comandos úteis para gerenciar serviços com `systemd`.

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Comandos básicos:

```bash
sudo systemctl start nome.service
sudo systemctl stop nome.service
sudo systemctl restart nome.service
sudo systemctl status nome.service
sudo systemctl enable nome.service   # iniciar no boot
sudo systemctl disable nome.service  # desativar no boot
```

Ver logs de serviço:

```bash
sudo journalctl -u nome.service
sudo journalctl -f                      # seguir logs em tempo real
```

Recarregar unidades após alterar arquivos de unidade:

```bash
sudo systemctl daemon-reload
sudo systemctl restart nome.service
```

Exemplo de unit file (`/etc/systemd/system/meu-servico.service`):

```ini
[Unit]
Description=Meu Serviço
After=network.target

[Service]
Type=simple
User=meuusuario
ExecStart=/usr/local/bin/meu-servico --flag
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Habilitar e iniciar:

```bash
sudo systemctl enable --now meu-servico.service
```

Dicas:
- Use `Restart=on-failure` para serviços críticos.
- Para debugging, execute o `ExecStart` manualmente com o mesmo ambiente.
- Use `systemctl list-units --type=service` para ver serviços ativos.
