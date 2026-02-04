# CPU - Monitoramento e informações (Linux)

Comandos úteis para observar uso e características da CPU.

## Índice

- [Conteúdo](#conteudo)

## <a id="conteudo">Conteúdo</a>

Ver informações da CPU:

```bash
lscpu
cat /proc/cpuinfo | grep "model name" -m 1
```

Monitoramento em tempo real:

```bash
top        # visão geral, pressione 1 para ver cada CPU
htop       # alternativa mais amigável (se instalada)
```

Estatísticas por CPU (sysstat):

```bash
mpstat -P ALL 1
```

Carga do sistema e interpretação:

- `uptime` mostra load average (1, 5, 15 minutos).
- Load > número de CPUs indica sobrecarga.

Identificar processos consumidores:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

Ferramentas adicionais:
- `perf` para profiling de alto nível.
- `powertop` para análise de consumo em laptops.

Dica: combine `top`/`htop` com `iotop` para distinguir uso de CPU vs I/O.
