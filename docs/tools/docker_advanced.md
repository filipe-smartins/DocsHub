# Docker Avançado — imagens, segurança e otimização

Este guia cobre temas avançados para construir, otimizar e operar containers Docker em produção.

## Multi-stage builds

- Use multi-stage builds para reduzir o tamanho da imagem e separar etapas de build e runtime.

```dockerfile
FROM python:3.11-slim as build
WORKDIR /app
COPY pyproject.toml setup.cfg ./
RUN pip install --upgrade pip && pip install build
COPY . .
RUN pip wheel --no-deps --wheel-dir /wheels .

FROM python:3.11-slim
COPY --from=build /wheels /wheels
RUN pip install --no-deps /wheels/*.whl
COPY . /app
CMD ["myapp"]
```

## Camadas e cache

- Ordene instruções para maximizar cache: arquivos raramente mudados (instalação de dependências) primeiro; cópias de código por último.
- Use `--cache-from` para builds em CI e `buildx` com cache remoto.

## BuildKit e `docker buildx`

- Habilite BuildKit para builds paralelos, cache avançado e `--mount=type=cache`.

## Redução de tamanho de imagem

- Use imagens base reduzidas (`slim`, `alpine` quando compatível).
- Remova ferramentas de build na etapa final (multi-stage).
- Minimize dependências e arquivos copiados (`.dockerignore`).

## Segurança de imagens

- Execute processos como usuário não-root (`USER 1000`).
- Assine e verifique imagens (Docker Content Trust/Notary, cosign).
- Escaneie imagens por vulnerabilidades (Trivy, Clair, Snyk) no pipeline.

## Volumes, dados e backups

- Prefira volumes nomeados para dados persistentes.
- Para bancos, use backups regulares fora do container; não confie em volumes locais para RTO/RPO críticos.

## Redes e isolamento

- Use redes bridge customizadas para isolar conjuntos de serviços.
- Para comunicação entre stacks, considere redes overlay (Swarm) ou CNI com Kubernetes.

## Healthchecks e readiness

- Utilize `HEALTHCHECK` nas imagens para que orquestradores e pipelines detectem instâncias saudáveis.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s CMD curl -f http://localhost:8080/health || exit 1
```

## Limites de recursos

- Configure `--memory` e `--cpus` no runtime ou orquestrador; não dependa apenas do cgroup padrão.

## Runtime e compatibilidade

- Teste imagens em runtimes distintos (Docker Engine, containerd) e em ambientes com SELinux/AppArmor.

## CI/CD e imagens

- Faça builds reprodutíveis (tags imutáveis) e publique em registry privado (registry, GCR, ECR, ACR).
- Use `latest` apenas para desenvolvimento; em produção use tags semânticas ou digestos (`@sha256:`).

## Observabilidade e logs

- Redirecione logs para stdout/stderr; agregue com Fluentd/Fluent Bit/Logstash.
- Exporte métricas via endpoint (Prometheus) e monitore containers em nível de host.

## Boas práticas rápidas

- Tenha `.dockerignore` para excluir arquivos desnecessários.
- Evite segredos em imagens; injete via runtime (secrets, env vars, vault).
- Recrie imagens e teste em staging antes de promover para produção.

Referências: documentação Docker, BuildKit, e guias dos scanners (Trivy, Snyk).
