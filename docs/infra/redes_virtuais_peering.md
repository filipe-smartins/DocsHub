# Redes Virtuais & Peering — conceitos e práticas

Resumo sobre redes virtuais em provedores cloud, peering entre redes e arquiteturas de conectividade.

## Redes Virtuais (VPC/VNet)

- VPC (AWS), VNet (Azure) ou rede privada em GCP isolam recursos em blocos CIDR privados.
- Segmente sub-redes por zona/propósito (public/private) e use routing/NAT para acesso à internet.

## Peering e conectividade

- **VPC Peering**: conecta duas VPCs para tráfego privado; é ponto-a-ponto e não roteia automaticamente para terceiros.
- **Transit Gateway / VPC Router**: soluções para conectar múltiplas VPCs via hub-and-spoke.
- **VPN / Direct Connect / ExpressRoute**: conectividade on-prem ↔ cloud com baixa latência/alta segurança.

## Network Policies e segurança

- Restrinja tráfego entre sub-redes com Security Groups / Network ACLs / NSGs.
- Use firewalls gerenciados e inspeção de tráfego para controlar fluxo L7.

## DNS e resolução entre redes

- Configure resolução privada (AWS Route53 Private Hosted Zones, Azure Private DNS) para nomes internos.

## Observabilidade e troubleshooting

- Monitore rotas, peerings e latência; use flow logs (VPC Flow Logs) para auditoria de tráfego.

## Boas práticas

- Planeje CIDR para evitar sobreposição ao conectar múltiplas redes.
- Prefira Transit Gateway/Hub quando houver múltiplos peerings para reduzir complexidade.
- Documente topology e políticas de rede; automatize com IaC (Terraform).

Referências: docs dos provedores cloud e guias de networking corporativo.
