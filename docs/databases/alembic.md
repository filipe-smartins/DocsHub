# Alembic — migrações de esquema em SQLAlchemy

Alembic é a ferramenta de migração usada com SQLAlchemy para versionar alterações de esquema (criação/alteração de tabelas, colunas, índices).

## Conceitos

- **Revision**: arquivo que descreve uma mudança (upgrade/downgrade).
- **Head**: versão mais recente.
- **Autogenerate**: Alembic pode gerar migrações comparando modelos SQLAlchemy com o banco.

## Comandos comuns

```bash
alembic init alembic                # inicializar diretório de migrações
alembic revision -m "create users" # criar nova revision vazia
alembic revision --autogenerate -m "add col"  # gerar a migração automaticamente
alembic upgrade head                # aplicar migrações
alembic downgrade -1                # reverter uma migração
alembic history --verbose           # ver histórico
```

## Estrutura de uma migration

- Arquivos em `alembic/versions/` com funções `upgrade()` e `downgrade()` que usam operações do `op` (op.create_table, op.add_column, etc.).

## Boas práticas

- Revise migrações autogeradas antes de aplicar—nem sempre detectam mudanças complexas.
- Evite alterações destrutivas em produção sem plano de migração de dados (ex.: renomear colunas, backfill em etapas).
- Use `op.bulk_insert` e transações quando possível para dados iniciais.
- Teste migrações em CI aplicando `alembic upgrade head` em uma cópia do DB.

## Considerações para deploys zero-downtime

- Escreva migrações compatíveis com deployment em duas fases: primeiro tornar código compatível com ambos os esquemas, depois remover campos antigos.
- Para grandes tabelas, evite `ALTER TABLE ...` que trava; use estratégias online (pg_repack, particionamento, adicionar coluna nova + backfill).

Referências: documentação Alembic e exemplos em projetos Flask/Django+SQLAlchemy.
