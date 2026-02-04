# Escaneamento de Dependências — segurança de supply chain

Ferramentas e práticas para detectar vulnerabilidades em dependências de projetos.

## Ferramentas comuns

- **pip-audit**: scanner de vulnerabilidades para pacotes Python.
- **Safety**: verificação de vulnerabilidades com base em banco de dados comercial/comunitário.
- **OWASP Dependency-Check**: suporte multi-linguagens (Java, JS, etc.).
- **Snyk / GitHub Dependabot**: automação de updates e alertas via integração com repositório.

## Integração com CI

- Execute scanners no pipeline e falhe o build em vulnerabilidades críticas.
- Combine com testes e análise estática para maior cobertura.

## Remediação

- Atualize dependências para versões seguras; se não for possível, aplique mitigations (patches, compensações).
- Use ferramentas que abrem PRs automáticos (Dependabot/Snyk) para manter deps atualizadas.

## Boas práticas

- Mantenha `requirements.txt`/`poetry.lock` atualizados e use hash-locking para reproducibilidade.
- Restrinja permissões de execução de código de dependências (ex.: scripts postinstall).
- Faça revisão humana de dependências novas ou não populares.

Referências: documentação pip-audit, Safety, Dependabot e Snyk.
