# PostgreSQL: instalação no Linux

Este guia cobre a instalação do PostgreSQL em distribuições Linux comuns (Debian/Ubuntu, RHEL/CentOS/Rocky/Alma e Fedora), além de passos básicos de pós-instalação.

## Índice

- [Pré-requisitos](#pre-requisitos)
- [Debian/Ubuntu](#debian-ubuntu)
- [RHEL/CentOS/Rocky/Alma](#rhel-centos-rocky-alma)
- [Fedora](#fedora)
- [Pós-instalação (recomendado)](#pos-instalacao-recomendado)
- [Verificação rápida](#verificacao-rapida)
- [Dicas](#dicas)

## <a id="pre-requisitos">Pré-requisitos</a>

- Acesso com privilégios de administrador (sudo).
- Conexão com a internet para baixar pacotes.

## <a id="debian-ubuntu">Debian/Ubuntu</a>

1. Atualize o índice de pacotes:

	sudo apt update

2. Instale o PostgreSQL e utilitários comuns:

	sudo apt install -y postgresql postgresql-contrib

3. Verifique o serviço:

	sudo systemctl status postgresql

4. Acesse o shell do PostgreSQL:

	sudo -i -u postgres
	psql

## <a id="rhel-centos-rocky-alma">RHEL/CentOS/Rocky/Alma</a>

1. Instale o repositório oficial do PostgreSQL (PGDG):

	sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

2. Desabilite o módulo padrão do PostgreSQL (se aplicável):

	sudo dnf -qy module disable postgresql

3. Instale o servidor e contribuições:

	sudo dnf install -y postgresql16-server postgresql16-contrib

4. Inicialize o banco de dados e habilite o serviço:

	sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
	sudo systemctl enable --now postgresql-16

5. Verifique o status:

	sudo systemctl status postgresql-16

## <a id="fedora">Fedora</a>

1. Instale o PostgreSQL:

	sudo dnf install -y postgresql-server postgresql-contrib

2. Inicialize o banco de dados e habilite o serviço:

	sudo postgresql-setup --initdb
	sudo systemctl enable --now postgresql

## <a id="pos-instalacao-recomendado">Pós-instalação (recomendado)</a>

### Alterar a senha do usuário postgres

1. Acesse o shell do PostgreSQL:

	sudo -i -u postgres
	psql

2. No prompt do psql, execute:

	ALTER USER postgres WITH PASSWORD 'sua_senha_segura';

### Criar um banco e um usuário

No psql:

- Criar usuário:

  CREATE USER app_user WITH PASSWORD 'senha_forte';

- Criar banco:

  CREATE DATABASE app_db OWNER app_user;

### Permitir conexões remotas (opcional)

1. Edite o arquivo postgresql.conf:

	listen_addresses = '*'

2. Edite o pg_hba.conf para liberar o IP ou a rede desejada, por exemplo:

	host    all             all             192.168.1.0/24          md5

3. Reinicie o serviço:

	sudo systemctl restart postgresql

## <a id="verificacao-rapida">Verificação rápida</a>

- Versão instalada:

  psql --version

- Conexão local:

  psql -U postgres -h localhost

## <a id="dicas">Dicas</a>

- Em ambientes de produção, prefira senhas fortes e firewall restritivo.
- Mantenha o PostgreSQL atualizado pelo gerenciador de pacotes do sistema.
