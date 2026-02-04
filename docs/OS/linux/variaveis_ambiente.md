# Variáveis de ambiente (Linux)

Variáveis de ambiente são usadas para configurar comportamento de processos sem editar código (ex.: `DATABASE_URL`, `SECRET_KEY`).

## Definir temporariamente (sessão atual)

```bash
export MINHA_VAR=valor
echo $MINHA_VAR
```

## Definir permanentemente (por usuário)

Adicione ao `~/.profile`, `~/.bashrc` ou `~/.bash_profile`:

```bash
export MINHA_VAR=valor
```

Depois reinicie sessão ou rode `source ~/.bashrc`.

## Arquivos `.env` e uso em aplicações

- Use arquivos `.env` para desenvolvimento local e bibliotecas como `python-dotenv` ou `django-environ` para carregá-los no startup.

Exemplo `.env`:

```
DATABASE_URL=postgres://user:pass@localhost:5432/db
SECRET_KEY=uma_chave_secreta
```

## Systemd e EnvironmentFile

Para serviços gerenciados por `systemd`, use `Environment` ou `EnvironmentFile` no unit file:

```ini
[Service]
Environment=MINHA_VAR=valor
EnvironmentFile=/etc/default/minha_app
ExecStart=/path/to/exec
```

## Segurança e boas práticas

- Nunca comite arquivos com segredos para o repositório.
- Use cofre de segredos (HashiCorp Vault, AWS Secrets Manager) em produção quando possível.
- Prefira permissões restritas em arquivos que contenham segredos (`chmod 600`).

Referências: man `env`, `systemd.exec` (EnvironmentFile).
