# SQL - DCL (Data Control Language)

DCL trata de permissões e controle de acesso: `GRANT` e `REVOKE`.

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Conceder permissões:

```sql
GRANT SELECT ON TABLE clientes TO usuario;
GRANT INSERT, UPDATE ON TABLE pedidos TO role_apis;
GRANT ALL PRIVILEGES ON DATABASE minha_db TO admin_user;
```

Revogar permissões:

```sql
REVOKE INSERT ON TABLE pedidos FROM role_apis;
REVOKE ALL PRIVILEGES ON DATABASE minha_db FROM admin_user;
```

Permissões em esquemas e funções:

```sql
GRANT USAGE ON SCHEMA financeiro TO relatórios;
GRANT EXECUTE ON FUNCTION processa_faturas() TO role_jobs;
```

Boas práticas:
- Use roles em vez de conceder diretamente a usuários individuais.
- Conceda o mínimo de privilégios necessários (principle of least privilege).
- Registre alterações e revise permissões periodicamente.

