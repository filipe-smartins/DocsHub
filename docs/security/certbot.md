# Certificados SSL — Certbot (Let's Encrypt)

Este guia mostra como obter e renovar certificados TLS gratuitos usando o Certbot (Let's Encrypt).

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Instalação (exemplos):

- Ubuntu/Debian (snap, recomendado):

```bash
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

- Debian/Ubuntu (apt, pacotes da distro):

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx  # para Nginx
```

Obter certificado usando integração automática (Nginx/Apache):

```bash
sudo certbot --nginx            # configura Nginx automaticamente
sudo certbot --apache           # para Apache
```

Obter certificado em modo standalone (útil se o servidor web não estiver rodando):

```bash
sudo systemctl stop nginx
sudo certbot certonly --standalone -d example.com -d www.example.com
sudo systemctl start nginx
```

Usar desafio DNS (para wildcard *.exemplo.com):

```bash
sudo certbot certonly --manual --preferred-challenges dns -d "*.example.com" -d example.com
```

Renovação automática:

- Certbot instalado via `snap` já cria um timer systemd que roda `certbot renew` regularmente.
- Para testar renovação manualmente (dry run):

```bash
sudo certbot renew --dry-run
```

Ver certificados e caminhos:

```bash
sudo certbot certificates
# certificados em /etc/letsencrypt/live/<domain>/
```

Integração com systemd (exemplo de service/timer se necessário):

```ini
[Unit]
Description=Certbot renewal

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot renew --deploy-hook "/bin/systemctl reload nginx"

[Install]
WantedBy=multi-user.target
```

Problemas comuns e dicas:
- Abra as portas 80 e 443 no firewall ao obter/validar certificados (Let's Encrypt usa porta 80 para HTTP-01 challenge).
- Use `--dry-run` antes de agendar mudanças em produção.
- Se usar DNS challenge, automatize com o plugin do provedor DNS (ex.: `certbot-dns-cloudflare`).
- Evite executar `certbot` repetidamente em curto intervalo (limits da CA).

Revogar e remover um certificado:

```bash
sudo certbot revoke --cert-path /etc/letsencrypt/live/example.com/cert.pem
sudo certbot delete --cert-name example.com
```

Referências rápidas:
- Certbot docs: https://certbot.eff.org/
- Let's Encrypt: https://letsencrypt.org/
