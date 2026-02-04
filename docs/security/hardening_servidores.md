# Hardening de Servidores — checklist essencial

Checklist prático para endurecer servidores Linux antes de entrar em produção.

## Atualizações e pacotes

- Mantenha sistema e pacotes atualizados (`apt upgrade` / `dnf upgrade`).
- Habilite atualizações automáticas de segurança quando apropriado.

## Acesso e contas

- Desabilite login SSH por senha: use `PasswordAuthentication no` e autenticação por chave.
- Restrinja usuários com `sudo` via `visudo` e use grupos, não usuários diretos.
- Remova contas/serviços desnecessários.

## SSH

- Use chaves ED25519/Ed448; limite `PermitRootLogin no`.
- Alterar porta não substitui regras de segurança, mas reduz ruído.

## Firewall e rede

- Configure firewall (ufw/iptables/nftables) bloqueando tudo por padrão e abrindo apenas portas necessárias.
- Use fail2ban para mitigar tentativas de brute-force.

## Proteções do kernel e confinamento

- Habilite SELinux ou AppArmor e valide perfis para serviços.
- Harden sysctl (ex.: `net.ipv4.ip_forward=0`, `net.ipv4.conf.all.rp_filter=1`).

## Minimizar superfície de ataque

- Execute apenas serviços necessários; use containers para isolar componentes quando adequado.
- Remova ferramentas de compilação em imagens de produção.

## Logs e auditoria

- Ative `auditd` para registros de segurança importantes.
- Centralize logs em um servidor/serviço de logs e monitore anomalias.

## Patching e backups

- Tenha processo de patch e rollback testado.
- Garanta backups e procedimentos de restore documentados.

## Secrets e configuração

- Não mantenha segredos em arquivos de configuração com controle de versão; use gerenciadores de segredo (Vault) ou environment vars seguras.

## Monitoramento e testes

- Monitore integridade, uso de recursos, e eventos de segurança (IDS/IPS quando aplicável).
- Faça pentests e exercícios de recuperação de incidente periodicamente.

Referências: guias CIS Benchmarks, documentação do fornecedor da distro.
