# Nginx - Guia de Referência

Comandos e configurações básicas para gerenciar o servidor Nginx.

## Índice

- [Gerenciamento do Serviço](#gerenciamento-do-servico)
- [Configuração e Testes](#configuracao-e-testes)
- [Logs](#logs)
- [Configuração Básica de Exemplo](#configuracao-basica-de-exemplo)

## <a id="gerenciamento-do-servico">Gerenciamento do Serviço</a>

- **Iniciar o Nginx:**
  ```bash
  sudo systemctl start nginx
  ```
- **Parar o Nginx:**
  ```bash
  sudo systemctl stop nginx
  ```
- **Reiniciar o Nginx:**
  ```bash
  sudo systemctl restart nginx
  ```
- **Recarregar configurações (sem queda):**
  ```bash
  sudo systemctl reload nginx
  ```
- **Verificar status:**
  ```bash
  sudo systemctl status nginx
  ```

## <a id="configuracao-e-testes">Configuração e Testes</a>

- **Testar sintaxe dos arquivos de config:**
  ```bash
  sudo nginx -t
  ```
- **Diretórios Padrão:**
  - Configurações: `/etc/nginx/`
  - Logs: `/var/log/nginx/`
  - Arquivos Web: `/var/www/html/`

## <a id="logs">Logs</a>

- **Ver logs de erro em tempo real:**
  ```bash
  sudo tail -f /var/log/nginx/error.log
  ```
- **Ver logs de acesso em tempo real:**
  ```bash
  sudo tail -f /var/log/nginx/access.log
  ```

## <a id="configuracao-basica-de-exemplo">Configuração Básica de Exemplo</a>

Localizado geralmente em `/etc/nginx/sites-available/default`:

```nginx
server {
    listen 80;
    server_name seu-dominio.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
