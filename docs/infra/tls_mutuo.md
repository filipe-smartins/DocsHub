# TLS Mútuo (mTLS) — autenticação cliente-serviço via certificados

mTLS adiciona autenticação do cliente ao fluxo TLS, garantindo que ambas as pontas se autentiquem mutuamente.

## Como funciona

- Tanto o servidor quanto o cliente apresentam certificados X.509 durante o handshake TLS.
- O servidor valida o certificado do cliente contra uma CA confiável (ou CRL/OCSP).

## Casos de uso

- Comunicação entre serviços em uma malha de serviço (service mesh) ou entre API gateway e serviços internos.
- Substitui tokens em cenários de alta segurança e proporciona autenticação baseada em identidade do certificado.

## Configuração básica (Nginx)

```nginx
ssl_client_certificate /etc/ssl/ca.crt;
ssl_verify_client on;

server {
  listen 443 ssl;
  ssl_certificate /etc/ssl/server.crt;
  ssl_certificate_key /etc/ssl/server.key;
}
```

## CA e rotação de certificados

- Use uma PKI interna (HashiCorp Vault PKI, step-ca, cert-manager) para emitir e revogar certificados.
- Planeje rotação automática e curto TTL para reduzir risco de comprometimento.

## Escalabilidade e gerenciamento

- Para muitos serviços, use automação (cert-manager em Kubernetes, Vault + Agent) para provisionar e renovar certificados.
- Integre com service mesh (Istio/Linkerd) que automatizam mTLS entre pods.

## Observabilidade e troubleshooting

- Registre o subject e serial do certificado nos logs de acesso para auditoria.
- Verifique erros de verificação de CA (cert chain issues) e CRL/OCSP.

## Boas práticas

- Enforce TLS1.2+ e ciphersuites preferindo TLS1.3 quando possível.
- Não confie apenas em certificados: combine com autorização (RBAC) para controlar recursos.

Referências: docs Nginx/HAProxy, HashiCorp Vault PKI, cert-manager.
