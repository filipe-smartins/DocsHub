# HAProxy / Nginx Avançado — balanceamento, TLS e tuning

Guia prático para configurar HAProxy e Nginx em ambientes de produção com foco em desempenho, TLS e resiliência.

## HAProxy — conceitos e configurações úteis

- *Mode* TCP vs HTTP: use TCP para passthrough (layer4) e HTTP para balanceamento baseado em cabeçalhos/URLs.
- *Health checks*: configure `option httpchk` para verificar endpoints de aplicação.
- *Stickiness*: use `cookie` ou `balance source` quando necessário manter sessão.
- *Timeouts*: ajuste `timeout connect`, `timeout client`, `timeout server` para evitar workers presos.
- *Load balancing algorithms*: `roundrobin`, `leastconn`, `source` — escolha conforme padrão de carga.

Exemplo minimal de frontend/backend HTTP:

```haproxy
frontend http-in
  bind *:80
  default_backend app

backend app
  balance leastconn
  server app1 10.0.0.1:8000 check
  server app2 10.0.0.2:8000 check
```

## Nginx — tuning e reverse proxy avançado

- Use `worker_processes auto;` e ajuste `worker_connections` de acordo com recursos.
- Optimize `keepalive_timeout`, `sendfile on`, `tcp_nopush on`, `tcp_nodelay on`.
- Cache estático e proxy_cache para reduzir carga do upstream.
- Configure `proxy_buffer_size` e `proxy_buffers` para responses grandes.

Exemplo de proxy com cache:

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:10m max_size=1g inactive=60m;
server {
  location /api/ {
    proxy_cache mycache;
    proxy_pass http://app_upstream;
  }
}
```

## TLS e offloading

- Terminar TLS no proxy (HAProxy/Nginx) reduz carga nos upstreams; para mTLS, HAProxy/Nginx suportam verificação de cliente.
- Use certificados com OCSP stapling (`ssl_stapling on;` em Nginx) e configure ciphersuites modernas (TLS1.2/1.3).

## Segurança e hardening

- Habilite HTTP Strict Transport Security (`add_header Strict-Transport-Security`), `X-Frame-Options`, `X-Content-Type-Options`.
- Limite body size (`client_max_body_size`) e proteja endpoints de upload.

## Observabilidade e métricas

- Exponha métricas Prometheus (HAProxy exporter, *nginx-vts-exporter* ou *nginx-prometheus-exporter*).
- Centralize logs (access + error) e use parsing para KPI (latência, 5xx rate).

## Escalabilidade e deploy

- Configure healthchecks, readiness probes e draining antes de remover instâncias.
- Para configuração dinâmica, use Consul + consul-template ou API do HAProxy (Runtime API) para reload sem downtime.

Referências: docs do HAProxy e Nginx, guias de performance.
