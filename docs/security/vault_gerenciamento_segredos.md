# Gerenciamento de Segredos — HashiCorp Vault

Visão prática sobre uso de Vault para armazenar e controlar acesso a segredos e chaves.

## Conceitos

- **Secret Engines**: KV (key/value), Transit (cryptography-as-a-service), PKI, etc.
- **Policies**: controle de acesso baseado em políticas (HCL) que limitam operações por caminho.
- **Auth Methods**: AppRole, Kubernetes, LDAP, Tokens — maneiras de autenticar clientes.

## Exemplo rápido — KV v2

```bash
# escrever segredo
vault kv put secret/app config='{"DB_URL":"..."}'

# ler segredo
vault kv get -format=json secret/app
```

## Uso em aplicações

- Autentique a aplicação (ex.: AppRole, Kubernetes auth) e busque segredos em runtime; não injetar tokens estáticos no código.
- Use Transit para criptografar dados sem expor chaves.

## Rotação e auditoria

- Automatize rotação de credenciais (DB, chaves) e registre acessos no audit log do Vault.

## Deployment e HA

- Rode Vault em modo HA com armazenamento persistente (Consul, AWS S3, etc.) e TLS obrigatório.

## Boas práticas

- Minimize permissões via policies, use leases TTL e revogue tokens comprometidos.
- Integre com CI/CD para injetar segredos dinamicamente em jobs.

Referências: docs HashiCorp Vault.
