# Administração Linux — Permissões

Resumo rápido sobre permissões de arquivos e mecanismos de controle.

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Permissões tradicionais (rwx) e representação numérica:

```text
-rwxr-xr--  => dono: rwx (7), grupo: r-x (5), others: r-- (4)  => 754
```

Comandos úteis:

```bash
ls -l arquivo
chmod 640 arquivo        # mudar permissões (numérico ou simbólico)
chown usuario:grupo arquivo
```

Setuid, setgid e sticky bit:

```bash
chmod u+s binario        # setuid (executa como dono)
chmod g+s diretorio      # setgid (novos arquivos herdam grupo)
chmod +t diretorio       # sticky bit (apenas dono apaga arquivos)
```

ACLs (acess control lists) para regras mais granulares:

```bash
getfacl arquivo
setfacl -m u:usuario:rwx arquivo   # adicionar permissão para usuário
setfacl -x u:usuario arquivo       # remover entrada ACL
```

Máscara padrão (umask):

```bash
umask                 # ver valor atual
umask 027             # exemplo: novos arquivos 750/640
```

Boas práticas:
- Evite usar `chmod 777` — é inseguro.
- Prefira grupos bem definidos para compartilhar recursos.
- Use ACLs quando as permissões POSIX não forem suficientes.
