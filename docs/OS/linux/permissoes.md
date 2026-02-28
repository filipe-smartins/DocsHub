# Administração Linux — Permissões

Guia completo sobre permissões de arquivos e diretórios no Linux, do nível básico ao avançado: permissões POSIX tradicionais, bits especiais, ACLs, atributos estendidos, capabilities, namespaces e boas práticas de segurança.

---

## Índice

- [Conceitos Fundamentais](#conceitos-fundamentais)
- [Permissões Tradicionais (rwx)](#permissoes-tradicionais)
- [Representação Numérica (Octal)](#representacao-numerica)
- [Comandos Essenciais — chmod, chown, chgrp](#comandos-essenciais)
- [chmod — Modo Simbólico vs Numérico](#chmod-detalhado)
- [chown e chgrp](#chown-chgrp)
- [Permissões em Diretórios](#permissoes-diretorios)
- [Bits Especiais — setuid, setgid e sticky bit](#bits-especiais)
- [Máscara Padrão — umask](#umask)
- [ACLs — Access Control Lists](#acls)
- [Atributos Estendidos — lsattr e chattr](#atributos-estendidos)
- [Capabilities — Alternativa ao setuid root](#capabilities)
- [Contextos SELinux e AppArmor](#selinux-apparmor)
- [Namespaces e Permissões em Contêineres](#namespaces)
- [Permissões em Sistemas de Arquivos Especiais](#fs-especiais)
- [Diagnóstico e Troubleshooting](#diagnostico)
- [Cenários Práticos](#cenarios-praticos)
- [Boas Práticas e Hardening](#boas-praticas)

---

## <a id="conceitos-fundamentais">Conceitos Fundamentais</a>

No Linux, **tudo é arquivo**: arquivos regulares, diretórios, dispositivos, sockets e pipes nomeados. Cada objeto no sistema de arquivos possui três camadas de metadados de acesso:

| Metadado | Descrição |
|----------|-----------|
| **UID** (User ID) | Identificador numérico do dono do arquivo |
| **GID** (Group ID) | Identificador numérico do grupo do arquivo |
| **Mode bits** | Conjunto de 12 bits que definem as permissões |

Quando um processo tenta acessar um arquivo, o kernel avalia na ordem:

1. Se o **UID efetivo** do processo é igual ao UID do arquivo → aplica permissões do **dono** (user).
2. Se o **GID efetivo** (ou GIDs suplementares) do processo corresponde ao GID do arquivo → aplica permissões do **grupo** (group).
3. Caso contrário → aplica permissões de **outros** (others).

> **Nota:** O usuário `root` (UID 0) ignora a maioria das verificações de permissão. A exceção é o bit de execução em arquivos regulares — root só executa se pelo menos um bit `x` estiver ativo (a não ser que use `CAP_DAC_OVERRIDE`).

### Inodes e Permissões

As permissões são armazenadas no **inode**, não no nome do arquivo. Isso significa que hard links compartilham as mesmas permissões, pois apontam para o mesmo inode:

```bash
ls -i arquivo              # mostrar número do inode
stat arquivo               # informações detalhadas do inode
```

Saída típica do `stat`:

```text
  File: arquivo.txt
  Size: 1024        Blocks: 8          IO Block: 4096   regular file
Device: 803h/2051d  Inode: 1234567     Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/   joao)   Gid: ( 1000/   joao)
Access: 2026-02-28 10:00:00.000000000 -0300
Modify: 2026-02-28 09:30:00.000000000 -0300
Change: 2026-02-28 09:30:00.000000000 -0300
 Birth: 2026-02-27 08:00:00.000000000 -0300
```

---

## <a id="permissoes-tradicionais">Permissões Tradicionais (rwx)</a>

Cada arquivo possui 9 bits de permissão divididos em três trios:

```text
 ┌── tipo do arquivo (d = diretório, - = regular, l = link, etc.)
 │
 │   ┌── dono (user)   ┌── grupo (group)   ┌── outros (others)
 │   │                  │                    │
 -   r w x              r - x                r - -
```

| Bit | Valor Numérico | Em Arquivos | Em Diretórios |
|-----|----------------|-------------|---------------|
| `r` (read) | 4 | Ler conteúdo | Listar conteúdo (`ls`) |
| `w` (write) | 2 | Modificar conteúdo | Criar/remover/renomear arquivos |
| `x` (execute) | 1 | Executar como programa | Acessar (entrar com `cd`) |

### Tipos de Arquivo (primeiro caractere)

| Caractere | Tipo |
|-----------|------|
| `-` | Arquivo regular |
| `d` | Diretório |
| `l` | Link simbólico |
| `c` | Dispositivo de caractere |
| `b` | Dispositivo de bloco |
| `p` | Pipe nomeado (FIFO) |
| `s` | Socket |

```bash
# Listar permissões detalhadas
ls -la /etc/passwd /tmp /dev/sda /dev/tty0

# Exemplo de saída:
# -rw-r--r--  1 root root   2847 Feb 28 10:00 /etc/passwd
# drwxrwxrwt 15 root root   4096 Feb 28 10:00 /tmp
# brw-rw----  1 root disk 8,  0 Feb 28 10:00 /dev/sda
# crw--w----  1 root tty  4,  0 Feb 28 10:00 /dev/tty0
```

---

## <a id="representacao-numerica">Representação Numérica (Octal)</a>

As permissões são representadas como um número octal de 3 ou 4 dígitos. Cada dígito é a soma dos bits ativos:

```text
r = 4    w = 2    x = 1

rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0
```

### Tabela de Referência Rápida

| Octal | Simbólico | Descrição |
|-------|-----------|-----------|
| `0` | `---` | Nenhuma permissão |
| `1` | `--x` | Apenas execução |
| `2` | `-w-` | Apenas escrita |
| `3` | `-wx` | Escrita e execução |
| `4` | `r--` | Apenas leitura |
| `5` | `r-x` | Leitura e execução |
| `6` | `rw-` | Leitura e escrita |
| `7` | `rwx` | Todas as permissões |

### Combinações Comuns

```text
755  =>  rwxr-xr-x   # executáveis, scripts, diretórios públicos
644  =>  rw-r--r--   # arquivos de configuração, documentos
700  =>  rwx------   # diretório home, chaves privadas
600  =>  rw-------   # arquivos sensíveis (chaves SSH, tokens)
750  =>  rwxr-x---   # diretório acessível apenas por dono e grupo
640  =>  rw-r-----   # arquivo legível pelo grupo, mas não por outros
444  =>  r--r--r--   # somente leitura para todos
```

### Exemplo Completo com 4 Dígitos (bits especiais)

```text
 ┌── bits especiais (setuid=4, setgid=2, sticky=1)
 │  ┌── dono
 │  │  ┌── grupo
 │  │  │  ┌── outros
 │  │  │  │
 4  7  5  5   =>  -rwsr-xr-x   (setuid ativo)
 2  7  5  5   =>  -rwxr-sr-x   (setgid ativo)
 1  7  7  7   =>  drwxrwxrwt   (sticky bit ativo)
```

---

## <a id="comandos-essenciais">Comandos Essenciais — chmod, chown, chgrp</a>

### <a id="chmod-detalhado">chmod — Alterar Permissões</a>

#### Modo Numérico (Octal)

```bash
chmod 755 script.sh          # rwxr-xr-x
chmod 644 config.yaml        # rw-r--r--
chmod 600 ~/.ssh/id_rsa      # rw------- (obrigatório para chaves SSH)
chmod 4755 /usr/bin/passwd   # setuid + rwxr-xr-x
```

#### Modo Simbólico

Sintaxe: `chmod [quem][operador][permissão] arquivo`

- **Quem:** `u` (user/dono), `g` (group), `o` (others), `a` (all = u+g+o)
- **Operador:** `+` (adicionar), `-` (remover), `=` (definir exatamente)
- **Permissão:** `r`, `w`, `x`, `s` (setuid/setgid), `t` (sticky), `X` (execução condicional)

```bash
chmod u+x script.sh          # adicionar execução para o dono
chmod g-w arquivo.txt         # remover escrita do grupo
chmod o= arquivo.txt          # remover todas as permissões de outros
chmod a+r documento.pdf       # adicionar leitura para todos
chmod u=rwx,g=rx,o= app      # definir permissões exatas (equivale a 750)
chmod ug+rw,o-rwx dados.csv   # dono e grupo leem/escrevem, outros nada
```

#### Permissão Condicional `X` (maiúsculo)

O `X` aplica execução **apenas** a diretórios e arquivos que já possuem algum bit de execução. Extremamente útil para aplicar permissões recursivamente sem tornar todos os arquivos executáveis:

```bash
# Corrigir permissões: diretórios 755, arquivos 644
chmod -R u=rwX,g=rX,o=rX /var/www/html

# Equivalente manual:
find /var/www/html -type d -exec chmod 755 {} \;
find /var/www/html -type f -exec chmod 644 {} \;
```

#### Opções do chmod

```bash
chmod -R 755 diretorio/           # recursivo
chmod -v 644 arquivo.txt          # verbose — mostra alterações
chmod -c 644 arquivo.txt          # como -v, mas só mostra se mudou
chmod --reference=modelo alvo     # copiar permissões de outro arquivo
chmod --preserve-root -R 755 /    # prevenir aplicação acidental em /
```

### <a id="chown-chgrp">chown e chgrp — Alterar Dono e Grupo</a>

```bash
# Alterar dono
chown joao arquivo.txt

# Alterar dono e grupo
chown joao:desenvolvedores arquivo.txt

# Alterar apenas o grupo (duas formas)
chown :desenvolvedores arquivo.txt
chgrp desenvolvedores arquivo.txt

# Recursivo
chown -R www-data:www-data /var/www/

# Copiar propriedade de outro arquivo
chown --reference=modelo alvo

# Alterar apenas se o dono atual corresponder (segurança)
chown --from=antigo_usuario novo_usuario arquivo.txt

# Não seguir links simbólicos (alterar o link, não o alvo)
chown -h joao:joao link_simbolico
```

> **Importante:** Apenas `root` pode alterar o dono de um arquivo. Usuários normais podem alterar o grupo apenas para grupos aos quais pertencem.

---

## <a id="permissoes-diretorios">Permissões em Diretórios</a>

As permissões em diretórios têm significados diferentes dos arquivos:

| Permissão | Efeito em Diretório |
|-----------|---------------------|
| `r` | Permite listar o conteúdo (`ls`) |
| `w` | Permite criar, renomear e remover arquivos dentro |
| `x` | Permite acessar (`cd`) e acessar inodes dos arquivos |

### Combinações Importantes

```bash
# Diretório com r mas sem x — pode listar nomes, mas não acessar metadados
chmod 744 dir/
ls dir/          # funciona (listagem parcial, sem detalhes)
cat dir/arq.txt  # FALHA (não consegue resolver o inode)

# Diretório com x mas sem r — pode acessar se souber o nome
chmod 711 dir/
ls dir/          # FALHA (não pode listar)
cat dir/arq.txt  # funciona (se souber o nome e o arquivo tiver permissão)

# Diretório com w sem sticky bit — qualquer um pode apagar qualquer arquivo
chmod 777 dir/
# Perigoso! Usuário A pode apagar arquivo de Usuário B
```

### Impacto na Criação de Arquivos

Para criar ou remover um arquivo de um diretório, o processo precisa de permissão `w` **e** `x` no diretório:

```bash
mkdir compartilhado
chmod 773 compartilhado/   # w+x para outros
# Outros podem criar e listar, mas veja o sticky bit para proteção
```

---

## <a id="bits-especiais">Bits Especiais — setuid, setgid e sticky bit</a>

### Setuid (Set User ID) — Bit 4000

Quando aplicado a um **executável**, o processo roda com o UID do **dono** do arquivo, não do usuário que executou:

```bash
chmod u+s programa          # ativar setuid
chmod 4755 programa         # setuid + rwxr-xr-x
ls -l programa
# -rwsr-xr-x  1 root root  ... programa
#    ^— "s" no lugar de "x" do dono
```

Se o dono **não** tem permissão de execução, aparece `S` (maiúsculo) — setuid sem efeito:

```bash
chmod 4655 programa
ls -l programa
# -rwSr-xr-x  (S maiúsculo = setuid sem x — inútil)
```

**Exemplos clássicos:**

```bash
ls -l /usr/bin/passwd /usr/bin/sudo /usr/bin/ping
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/sudo
# -rwsr-xr-x 1 root root ... /usr/bin/ping
```

> **Segurança:** Binários setuid são vetores de escalação de privilégio. Audite-os regularmente.

```bash
# Encontrar todos os binários setuid no sistema
find / -perm -4000 -type f 2>/dev/null

# Encontrar setuid pertencentes ao root
find / -perm -4000 -user root -type f 2>/dev/null
```

### Setgid (Set Group ID) — Bit 2000

Em **executáveis**, o processo roda com o GID do grupo do arquivo.

Em **diretórios**, novos arquivos e subdiretórios criados herdam o grupo do diretório pai (em vez do grupo primário do criador):

```bash
# Em executável
chmod g+s programa
chmod 2755 programa
ls -l programa
# -rwxr-sr-x  1 root staff  ... programa

# Em diretório (uso mais comum)
mkdir projeto
chown :desenvolvedores projeto
chmod 2775 projeto
# Agora, qualquer arquivo criado em "projeto" pertencerá ao grupo "desenvolvedores"

touch projeto/novo.txt
ls -l projeto/novo.txt
# -rw-rw-r-- 1 joao desenvolvedores ... novo.txt
```

### Sticky Bit — Bit 1000

Aplicado a **diretórios**, impede que usuários apaguem ou renomeiem arquivos de outros usuários, mesmo que tenham permissão de escrita no diretório.

O exemplo clássico é `/tmp`:

```bash
ls -ld /tmp
# drwxrwxrwt 15 root root 4096 ... /tmp
#          ^— "t" no lugar de "x" de others

chmod +t diretorio
chmod 1777 diretorio
```

Se others **não** tem permissão de execução, aparece `T` (maiúsculo):

```bash
chmod 1776 diretorio
ls -ld diretorio
# drwxrwxrwT  (T maiúsculo = sticky sem x para others)
```

### Resumo dos Bits Especiais

| Bit | Valor | Simbólico | Em Arquivo | Em Diretório |
|-----|-------|-----------|------------|--------------|
| Setuid | 4000 | `u+s` | Executa como dono | Sem efeito no Linux |
| Setgid | 2000 | `g+s` | Executa como grupo | Novos arquivos herdam grupo |
| Sticky | 1000 | `+t` | Sem efeito no Linux | Apenas dono pode apagar |

---

## <a id="umask">Máscara Padrão — umask</a>

O `umask` define quais permissões são **removidas** ao criar novos arquivos e diretórios.

### Como Funciona

```text
Permissão padrão de arquivos:    666 (rw-rw-rw-)
Permissão padrão de diretórios:  777 (rwxrwxrwx)

Permissão efetiva = padrão AND NOT(umask)

Exemplo com umask 022:
  Arquivos:     666 AND NOT(022) = 644 (rw-r--r--)
  Diretórios:   777 AND NOT(022) = 755 (rwxr-xr-x)

Exemplo com umask 027:
  Arquivos:     666 AND NOT(027) = 640 (rw-r-----)
  Diretórios:   777 AND NOT(027) = 750 (rwxr-x---)

Exemplo com umask 077:
  Arquivos:     666 AND NOT(077) = 600 (rw-------)
  Diretórios:   777 AND NOT(077) = 700 (rwx------)
```

### Comandos

```bash
umask                  # exibir umask atual (octal)
umask -S               # exibir em formato simbólico (u=rwx,g=rx,o=rx)
umask 027              # definir umask para a sessão atual
umask u=rwx,g=rx,o=    # formato simbólico (equivale a 027)
```

### Persistência do umask

O umask é definido por sessão. Para persistir:

```bash
# Para um usuário específico — ~/.bashrc ou ~/.profile
echo "umask 027" >> ~/.bashrc

# Para todos os usuários — /etc/profile ou /etc/login.defs
echo "umask 027" >> /etc/profile

# Em /etc/login.defs (para login sessions)
UMASK 027

# Para serviços systemd
# No arquivo .service:
# [Service]
# UMask=0027
```

### Tabela Prática de umask

| umask | Arquivo | Diretório | Uso típico |
|-------|---------|-----------|------------|
| `000` | 666 (rw-rw-rw-) | 777 (rwxrwxrwx) | Nenhuma restrição (inseguro) |
| `022` | 644 (rw-r--r--) | 755 (rwxr-xr-x) | Padrão da maioria das distros |
| `027` | 640 (rw-r-----) | 750 (rwxr-x---) | Servidores, multi-usuário |
| `077` | 600 (rw-------) | 700 (rwx------) | Máxima privacidade |
| `002` | 664 (rw-rw-r--) | 775 (rwxrwxr-x) | Trabalho colaborativo em grupo |

---

## <a id="acls">ACLs — Access Control Lists</a>

ACLs POSIX estendem o modelo clássico dono/grupo/outros, permitindo definir permissões para **múltiplos usuários e grupos** em um mesmo arquivo.

### Pré-requisitos

O sistema de arquivos precisa suportar ACLs (ext4, xfs, btrfs suportam nativamente). Verifique:

```bash
# Verificar se o FS foi montado com suporte a ACL
mount | grep acl
tune2fs -l /dev/sda1 | grep "Default mount options"

# Instalar ferramentas (se necessário)
sudo apt install acl        # Debian/Ubuntu
sudo dnf install acl        # RHEL/Fedora
```

### Visualizar ACLs

```bash
getfacl arquivo.txt
```

Saída:

```text
# file: arquivo.txt
# owner: joao
# group: desenvolvedores
user::rw-
user:maria:rw-          # ACL específica para maria
group::r--
group:design:r-x        # ACL específica para grupo design
mask::rwx
other::---
```

### Definir ACLs

```bash
# Adicionar permissão para um usuário
setfacl -m u:maria:rw arquivo.txt

# Adicionar permissão para um grupo
setfacl -m g:design:rx arquivo.txt

# Múltiplas regras de uma vez
setfacl -m u:maria:rw,g:design:rx,o::- arquivo.txt

# Remover ACL de um usuário
setfacl -x u:maria arquivo.txt

# Remover ACL de um grupo
setfacl -x g:design arquivo.txt

# Remover TODAS as ACLs (voltar ao modo POSIX simples)
setfacl -b arquivo.txt

# Aplicar recursivamente
setfacl -R -m u:maria:rx /var/www/html/
```

### ACLs Padrão (Default ACLs) em Diretórios

ACLs padrão são herdadas por novos arquivos e subdiretórios criados dentro:

```bash
# Definir ACL padrão no diretório
setfacl -d -m u:maria:rwx /projeto/
setfacl -d -m g:desenvolvedores:rwx /projeto/

# Verificar ACLs padrão
getfacl /projeto/
```

Saída com ACLs padrão:

```text
# file: projeto/
# owner: joao
# group: desenvolvedores
user::rwx
group::r-x
other::---
default:user::rwx
default:user:maria:rwx
default:group::r-x
default:group:desenvolvedores:rwx
default:mask::rwx
default:other::---
```

### A Máscara ACL (mask)

A máscara limita a permissão **efetiva** de todas as entradas ACL (exceto dono e others):

```bash
# Definir máscara — limita permissões efetivas de grupo e ACLs de usuário
setfacl -m m::rx arquivo.txt

# Verificar permissões efetivas
getfacl arquivo.txt
# user:maria:rwx       #effective:r-x   ← limitado pela máscara
```

> **Atenção:** O `chmod` altera a máscara ACL quando o arquivo tem ACLs, podendo restringir permissões inesperadamente.

### Indicador de ACL no ls

Quando um arquivo tem ACLs, o `ls -l` mostra um `+` após as permissões:

```bash
ls -l arquivo.txt
# -rw-rwx---+ 1 joao desenvolvedores 1024 ... arquivo.txt
#           ^— indica ACLs ativas
```

### Backup e Restauração de ACLs

```bash
# Exportar ACLs de uma árvore de diretórios
getfacl -R /projeto > acls_backup.txt

# Restaurar ACLs
setfacl --restore=acls_backup.txt
```

---

## <a id="atributos-estendidos">Atributos Estendidos — lsattr e chattr</a>

Atributos estendidos oferecem controle adicional **além** das permissões POSIX. São manipulados com `chattr` (change attribute) e `lsattr` (list attributes).

### Visualizar Atributos

```bash
lsattr arquivo.txt
# ----i--------e-- arquivo.txt
#     ^— imutável

lsattr -d /diretorio/     # atributos do diretório em si
lsattr -R /diretorio/     # recursivo
```

### Atributos Principais

| Atributo | Letra | Descrição |
|----------|-------|-----------|
| Imutável | `i` | Não pode ser modificado, renomeado, removido, nem ter links. Nem root consegue alterar (sem remover o atributo antes) |
| Append-only | `a` | Só permite adicionar dados (append). Ideal para logs |
| No dump | `d` | Ignorado pelo `dump` (backup) |
| Sync | `S` | Escritas são sincronizadas imediatamente no disco |
| No atime | `A` | Não atualiza o timestamp de acesso (performance) |
| Compressed | `c` | Compressão transparente (se suportado pelo FS) |
| Undeletable | `u` | Conteúdo é salvo ao deletar, permitindo recuperação |
| Secure delete | `s` | Ao deletar, os blocos são zerados |

### Exemplos Práticos

```bash
# Tornar arquivo imutável (nem root pode alterar!)
sudo chattr +i /etc/resolv.conf

# Tentar alterar (falha)
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
# bash: /etc/resolv.conf: Operation not permitted

# Remover imutável
sudo chattr -i /etc/resolv.conf

# Modo append-only para logs
sudo chattr +a /var/log/auditoria.log
echo "entrada de log" >> /var/log/auditoria.log   # funciona
echo "sobrescrever" > /var/log/auditoria.log       # FALHA

# Desativar atualização de atime (performance em SSDs)
sudo chattr +A /dados/leitura_intensiva/
```

> **Segurança:** `chattr +i` é útil para proteger arquivos de configuração críticos contra modificações acidentais ou maliciosas.

---

## <a id="capabilities">Capabilities — Alternativa ao setuid root</a>

As capabilities dividem os privilégios do root em unidades granulares, evitando conceder poder total via setuid:

### Por que Usar Capabilities

Em vez de:

```bash
chmod u+s /usr/bin/meu_servidor   # setuid root (inseguro — dá TODOS os poderes de root)
```

Prefira conceder apenas a capability necessária:

```bash
# Permitir que o programa faça bind em portas < 1024 sem ser root
sudo setcap 'cap_net_bind_service=+ep' /usr/bin/meu_servidor
```

### Capabilities Comuns

| Capability | Descrição |
|------------|-----------|
| `CAP_NET_BIND_SERVICE` | Bind em portas privilegiadas (< 1024) |
| `CAP_NET_RAW` | Usar RAW sockets (ping, tcpdump) |
| `CAP_DAC_OVERRIDE` | Ignorar verificações de permissão r/w/x |
| `CAP_DAC_READ_SEARCH` | Ignorar verificação de leitura e busca em diretórios |
| `CAP_CHOWN` | Alterar UIDs e GIDs de arquivos |
| `CAP_SYS_ADMIN` | Diversas operações administrativas (mount, etc.) |
| `CAP_SYS_PTRACE` | Depurar processos de outros usuários |
| `CAP_SETUID` | Manipular UIDs de processos |
| `CAP_KILL` | Enviar sinais para processos de outros usuários |
| `CAP_SYS_TIME` | Alterar relógio do sistema |

### Comandos

```bash
# Verificar capabilities de um executável
getcap /usr/bin/ping
# /usr/bin/ping = cap_net_raw+ep

# Listar todas as capabilities em um caminho
getcap -r /usr/bin/ 2>/dev/null

# Definir capability
sudo setcap 'cap_net_bind_service=+ep' /opt/app/servidor

# Remover todas as capabilities
sudo setcap -r /opt/app/servidor

# Ver capabilities do processo em execução
cat /proc/<PID>/status | grep -i cap
getpcaps <PID>

# Decodificar capabilities numéricas
capsh --decode=00000000a80425fb
```

### Flags das Capabilities

| Flag | Significado |
|------|------------|
| `e` (effective) | Capability está ativa |
| `p` (permitted) | Capability pode ser ativada |
| `i` (inheritable) | Capability é herdada por processos filhos |

```bash
# ep = effective + permitted (mais comum)
sudo setcap 'cap_net_bind_service=+ep' /opt/app/servidor

# eip = effective + inheritable + permitted
sudo setcap 'cap_net_bind_service=+eip' /opt/app/servidor
```

---

## <a id="selinux-apparmor">Contextos SELinux e AppArmor</a>

Sistemas MAC (Mandatory Access Control) adicionam uma camada de segurança **acima** das permissões POSIX.

### SELinux (Security-Enhanced Linux)

Usado por padrão no RHEL, CentOS, Fedora. Cada arquivo e processo possui um **contexto de segurança**:

```bash
# Ver contexto SELinux de arquivos
ls -lZ /var/www/html/
# -rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html

# Ver contexto de processos
ps auxZ | grep httpd

# Verificar modo atual do SELinux
getenforce          # Enforcing, Permissive, Disabled
sestatus            # informações detalhadas
```

Comandos principais:

```bash
# Alterar contexto de arquivo
sudo chcon -t httpd_sys_content_t /var/www/html/novo.html

# Restaurar contexto padrão (baseado na política)
sudo restorecon -Rv /var/www/html/

# Definir contexto padrão permanente
sudo semanage fcontext -a -t httpd_sys_content_t "/dados/web(/.*)?"
sudo restorecon -Rv /dados/web/

# Verificar logs de negação
sudo ausearch -m avc -ts recent
sudo sealert -a /var/log/audit/audit.log

# Booleanos SELinux (ajustes rápidos de política)
getsebool -a | grep httpd
sudo setsebool -P httpd_can_network_connect on
```

### AppArmor

Usado por padrão no Ubuntu, SUSE. Usa perfis baseados em caminhos:

```bash
# Ver status
sudo aa-status

# Modos de perfil
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx    # modo enforcing
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx   # modo aprendizado
sudo aa-disable /etc/apparmor.d/usr.sbin.nginx    # desabilitar perfil

# Recarregar perfis
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx

# Gerar perfil automaticamente
sudo aa-genprof /usr/bin/meu_app

# Logs de negação
sudo dmesg | grep apparmor
journalctl | grep apparmor
```

---

## <a id="namespaces">Namespaces e Permissões em Contêineres</a>

Contêineres (Docker, Podman, LXC) usam **user namespaces** para mapear UIDs/GIDs do contêiner para UIDs/GIDs diferentes no host.

### User Namespaces

```bash
# Verificar mapeamento de UID dentro de um contêiner
cat /proc/self/uid_map
#          0       1000          1
# UID 0 (root) no contêiner → UID 1000 no host

# Executar contêiner com user namespace (rootless)
podman run --userns=auto -it alpine id
# uid=0(root) gid=0(root)   ← root DENTRO do contêiner, sem privilégio real
```

### Permissões em Volumes Docker

```bash
# Problema comum: permissão negada em volumes montados
docker run -v /dados:/app/dados minha_imagem
# O UID dentro do contêiner pode não ter permissão nos arquivos do host

# Soluções:
# 1. Ajustar UID no Dockerfile
# RUN useradd -u 1000 appuser
# USER appuser

# 2. Usar --user para definir UID/GID
docker run --user 1000:1000 -v /dados:/app/dados minha_imagem

# 3. fixar permissões no host
chown -R 1000:1000 /dados/
```

### Rootless Containers

```bash
# Verificar suporte
sysctl kernel.unprivileged_userns_clone
# kernel.unprivileged_userns_clone = 1

# Limites de sub-UIDs e sub-GIDs
cat /etc/subuid
# joao:100000:65536     ← joao pode mapear 65536 UIDs a partir de 100000

cat /etc/subgid
# joao:100000:65536
```

---

## <a id="fs-especiais">Permissões em Sistemas de Arquivos Especiais</a>

### /proc e /sys

```bash
# Permissões em /proc controlam acesso a informações de processos
ls -l /proc/1/
# Muitos arquivos são legíveis apenas pelo dono do processo ou root

# hidepid — ocultar processos de outros usuários
# Em /etc/fstab:
# proc  /proc  proc  defaults,hidepid=2  0  0
# hidepid=0: padrão (todos veem todos os processos)
# hidepid=1: usuários não veem /proc/<PID>/ de outros
# hidepid=2: /proc/<PID>/ é invisível para outros usuários
mount -o remount,hidepid=2 /proc
```

### tmpfs

```bash
# Monterr tmpfs com permissões específicas
mount -t tmpfs -o size=100M,mode=1770,uid=0,gid=1000 tmpfs /tmp/seguro
```

### Opções de Montagem Relacionadas a Permissões

```bash
# Em /etc/fstab — opções de segurança
/dev/sdb1  /dados  ext4  defaults,nosuid,noexec,nodev  0  2

# nosuid   → ignora setuid/setgid bits (segurança)
# noexec   → proíbe execução de binários (proteção em /tmp, /var/tmp)
# nodev    → ignora dispositivos especiais
# ro       → montagem somente leitura
```

Verificar flags atuais:

```bash
mount | grep /dados
findmnt -o TARGET,OPTIONS /dados
cat /proc/mounts | grep /dados
```

---

## <a id="diagnostico">Diagnóstico e Troubleshooting</a>

### Verificar Permissões Efetivas

```bash
# Testar se um usuário pode acessar um arquivo
sudo -u maria test -r /dados/relatorio.csv && echo "Pode ler" || echo "Sem acesso"
sudo -u maria test -w /dados/relatorio.csv && echo "Pode escrever" || echo "Sem acesso"
sudo -u maria test -x /opt/scripts/deploy.sh && echo "Pode executar" || echo "Sem acesso"

# Verificar acesso completo (leitura, escrita, execução)
sudo -u maria namei -l /dados/relatorio.csv
# Mostra as permissões de CADA componente do caminho
```

### namei — Diagnóstico de Caminho Completo

```bash
namei -l /var/www/html/index.html
```

Saída:

```text
f: /var/www/html/index.html
drwxr-xr-x root     root     /
drwxr-xr-x root     root     var
drwxr-xr-x root     root     www
drwxr-xr-x www-data www-data html
-rw-r--r-- www-data www-data index.html
```

> **Dica:** Se qualquer diretório no caminho não tiver `x`, o acesso ao arquivo será negado, mesmo que o arquivo em si tenha permissões abertas.

### Auditoria com find

```bash
# Arquivos com permissão 777 (inseguros)
find / -perm 777 -type f 2>/dev/null

# Arquivos setuid
find / -perm -4000 -type f 2>/dev/null

# Arquivos setgid
find / -perm -2000 -type f 2>/dev/null

# Arquivos sem dono (órfãos — possível problema de segurança)
find / -nouser -o -nogroup 2>/dev/null

# Arquivos graváveis por todos (world-writable)
find / -perm -o+w -type f ! -path "/proc/*" ! -path "/sys/*" 2>/dev/null

# Diretórios graváveis sem sticky bit
find / -perm -o+w -type d ! -perm -1000 2>/dev/null

# Arquivos modificados nas últimas 24h (detecção de intrusão)
find / -mtime -1 -type f ! -path "/proc/*" ! -path "/sys/*" 2>/dev/null
```

### strace — Rastrear Chamadas de Sistema

Quando um programa falha com "Permission denied" e a causa não é óbvia:

```bash
strace -e trace=open,openat,access,stat -f programa 2>&1 | grep -i "permission\|EACCES\|EPERM"
```

### Logs de Auditoria (auditd)

```bash
# Monitorar acesso a um arquivo específico
sudo auditctl -w /etc/shadow -p rwa -k shadow_access

# Ver eventos
sudo ausearch -k shadow_access

# Monitorar mudanças de permissão
sudo auditctl -w /etc/ -p a -k etc_changes
```

---

## <a id="cenarios-praticos">Cenários Práticos</a>

### 1. Servidor Web (Nginx/Apache)

```bash
# Dono: www-data (processo do servidor)
sudo chown -R www-data:www-data /var/www/html/

# Diretórios: 755, Arquivos: 644
sudo find /var/www/html/ -type d -exec chmod 755 {} \;
sudo find /var/www/html/ -type f -exec chmod 644 {} \;

# Upload directory: servidor precisa escrever
sudo chmod 775 /var/www/html/uploads/

# PHP/scripts que não devem ser executados via web
sudo chmod 640 /var/www/html/config.php

# Bloquear execução em diretório de uploads (via nginx)
# location /uploads/ { deny all; }
```

### 2. Diretório de Projeto Compartilhado

```bash
# Criar grupo e adicionar usuários
sudo groupadd projeto-alpha
sudo usermod -aG projeto-alpha joao
sudo usermod -aG projeto-alpha maria
sudo usermod -aG projeto-alpha pedro

# Criar diretório com setgid
sudo mkdir /opt/projeto-alpha
sudo chown root:projeto-alpha /opt/projeto-alpha
sudo chmod 2775 /opt/projeto-alpha

# ACL padrão para novos arquivos (opcional — garante herança)
sudo setfacl -d -m g:projeto-alpha:rwx /opt/projeto-alpha

# umask dos usuários deve permitir grupo (ex: 002 ou 007)
```

### 3. Chaves SSH — Permissões Obrigatórias

O OpenSSH recusa funcionar se as permissões não estiverem corretas:

```bash
chmod 700 ~/.ssh                  # diretório .ssh
chmod 600 ~/.ssh/id_rsa           # chave privada
chmod 644 ~/.ssh/id_rsa.pub       # chave pública
chmod 600 ~/.ssh/authorized_keys  # chaves autorizadas
chmod 644 ~/.ssh/known_hosts      # hosts conhecidos
chmod 644 ~/.ssh/config           # configuração do cliente

# O diretório home também não pode ser gravável por grupo/outros
chmod 755 ~
# ou
chmod 750 ~
```

### 4. Banco de Dados PostgreSQL

```bash
# Diretório de dados — apenas o usuário postgres
sudo chmod 700 /var/lib/postgresql/16/main/
sudo chown postgres:postgres /var/lib/postgresql/16/main/

# Arquivos de certificado TLS
chmod 600 /etc/postgresql/16/main/server.key
chmod 644 /etc/postgresql/16/main/server.crt
chown postgres:postgres /etc/postgresql/16/main/server.*
```

### 5. Proteger Configurações Críticas

```bash
# Imutável — nem root modifica sem remover o atributo
sudo chattr +i /etc/passwd
sudo chattr +i /etc/shadow
sudo chattr +i /etc/group
sudo chattr +i /etc/sudoers
sudo chattr +i /etc/ssh/sshd_config

# Para editar, remover temporariamente:
sudo chattr -i /etc/ssh/sshd_config
sudo vim /etc/ssh/sshd_config
sudo chattr +i /etc/ssh/sshd_config
```

### 6. Script de Correção de Permissões

```bash
#!/bin/bash
# fix-permissions.sh — Corrigir permissões de um diretório web
set -euo pipefail

WEBROOT="${1:?Uso: $0 /caminho/do/webroot}"
WEB_USER="www-data"
WEB_GROUP="www-data"

echo "Ajustando dono para ${WEB_USER}:${WEB_GROUP}..."
chown -R "${WEB_USER}:${WEB_GROUP}" "${WEBROOT}"

echo "Ajustando diretórios para 755..."
find "${WEBROOT}" -type d -exec chmod 755 {} \;

echo "Ajustando arquivos para 644..."
find "${WEBROOT}" -type f -exec chmod 644 {} \;

echo "Ajustando scripts para 750..."
find "${WEBROOT}" -name "*.sh" -exec chmod 750 {} \;

echo "Permissões corrigidas com sucesso."
```

---

## <a id="boas-praticas">Boas Práticas e Hardening</a>

### Princípios Fundamentais

1. **Princípio do menor privilégio** — conceda apenas as permissões estritamente necessárias.
2. **Nunca use `chmod 777`** — se precisa de acesso compartilhado, use grupos ou ACLs.
3. **Evite executar serviços como root** — use `User=` em unidades systemd.
4. **Audite binários setuid regularmente** — são vetores de escalação.
5. **Use capabilities em vez de setuid root** quando possível.

### Checklist de Segurança

```bash
# 1. Encontrar e revisar binários setuid/setgid
find / -perm /6000 -type f 2>/dev/null | sort

# 2. Encontrar arquivos world-writable
find / -xdev -perm -o+w ! -type l -type f 2>/dev/null

# 3. Encontrar diretórios world-writable sem sticky bit
find / -xdev -perm -o+w -type d ! -perm -1000 2>/dev/null

# 4. Encontrar arquivos sem dono
find / -xdev -nouser -o -nogroup 2>/dev/null

# 5. Verificar permissões de /etc/shadow
ls -l /etc/shadow    # deve ser 640 root:shadow ou 000 root:root

# 6. Verificar opções de montagem
mount | grep -E "nosuid|noexec|nodev"
# /tmp e /var/tmp devem ter nosuid,noexec,nodev

# 7. Verificar umask padrão
grep -i umask /etc/login.defs /etc/profile /etc/bash.bashrc 2>/dev/null
```

### Permissões Recomendadas para Arquivos Críticos

| Arquivo/Diretório | Permissão | Dono | Notas |
|--------------------|-----------|------|-------|
| `/etc/passwd` | 644 | root:root | Legível por todos (sem senhas) |
| `/etc/shadow` | 640 | root:shadow | Hashes de senha — restrito |
| `/etc/sudoers` | 440 | root:root | Editar apenas com `visudo` |
| `/etc/ssh/sshd_config` | 600 | root:root | Configuração do daemon SSH |
| `~/.ssh/` | 700 | user:user | Diretório de chaves SSH |
| `~/.ssh/id_rsa` | 600 | user:user | Chave privada |
| `~/.ssh/authorized_keys` | 600 | user:user | Chaves autorizadas |
| `/var/log/` | 755 | root:syslog | Diretório de logs |
| `/tmp` | 1777 | root:root | Sticky bit obrigatório |
| `/var/tmp` | 1777 | root:root | Sticky bit obrigatório |
| Certificados TLS (`.key`) | 600 | root:root | Chave privada do certificado |
| Certificados TLS (`.crt`) | 644 | root:root | Certificado público |

### Endurecimento de Partições (/etc/fstab)

```text
# Exemplo de /etc/fstab com opções de segurança
/dev/sda2   /tmp      ext4  defaults,nosuid,noexec,nodev  0  2
/dev/sda3   /var/tmp  ext4  defaults,nosuid,noexec,nodev  0  2
/dev/sda4   /var/log  ext4  defaults,nosuid,noexec,nodev  0  2
/dev/sda5   /home     ext4  defaults,nosuid,nodev         0  2
```

### Automação com Ansible (Exemplo)

```yaml
# Garantir permissões corretas em servidores
- name: Ajustar permissões de /etc/shadow
  ansible.builtin.file:
    path: /etc/shadow
    owner: root
    group: shadow
    mode: '0640'

- name: Garantir sticky bit em /tmp
  ansible.builtin.file:
    path: /tmp
    mode: '1777'
    state: directory

- name: Remover permissões excessivas
  ansible.builtin.command:
    cmd: find /var/www -perm -o+w -type f -exec chmod o-w {} \;
  changed_when: false
```

### Monitoramento Contínuo com auditd

```bash
# /etc/audit/rules.d/permissions.rules

# Monitorar alterações de permissão em arquivos críticos
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/sudoers -p wa -k sudoers

# Monitorar uso de chmod/chown
-a always,exit -F arch=b64 -S chmod,fchmod,fchmodat -F auid>=1000 -k perm_mod
-a always,exit -F arch=b64 -S chown,fchown,fchownat,lchown -F auid>=1000 -k owner_mod

# Monitorar uso de setxattr (ACLs e atributos estendidos)
-a always,exit -F arch=b64 -S setxattr,lsetxattr,fsetxattr -F auid>=1000 -k xattr_mod
```

---

**Referências:**

- `man chmod`, `man chown`, `man chgrp`, `man umask`
- `man setfacl`, `man getfacl`
- `man chattr`, `man lsattr`
- `man capabilities`, `man setcap`, `man getcap`
- `man mount` (opções nosuid, noexec, nodev)
- [Linux man-pages — credentials(7)](https://man7.org/linux/man-pages/man7/credentials.7.html)
- [Linux man-pages — namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
