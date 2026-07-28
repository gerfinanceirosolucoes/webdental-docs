# Infraestrutura

Conta AWS `843630347602`. Staging em `sa-east-1`, produção em `us-east-1` (VPC `vpc-81f5f4f8`).

## Novos recursos — ALB Internal (Julho/2026)

### Staging (sa-east-1)

| Recurso | Nome / Valor |
|---|---|
| ALB Internal | `hmg-mevo-alb-internal` |
| DNS | `internal-hmg-mevo-alb-internal-1641981751.sa-east-1.elb.amazonaws.com` |
| Security Group | `hmg-mevo-alb-internal-sg` — ingress TCP 80 de `172.31.0.0/16` |
| Target Group | `hmg-mevo-internal-target-group` — HTTP, porta 3000, health check `/health` |
| SSM Parameter | `/mevo-api/staging/internal-allowed-ips` → `172.31.0.0/16` |
| Secret | `hmg/mevo-api/internal-api-key` (Secrets Manager) |

### Produção (us-east-1)

| Recurso | Nome / Valor |
|---|---|
| ALB Internal | `prd-mevo-alb-internal` |
| DNS | `internal-prd-mevo-alb-internal-1961577618.us-east-1.elb.amazonaws.com` |
| Security Group | `prd-mevo-alb-internal-sg` — ingress TCP 80 de `172.31.0.0/16` |
| Target Group | `prd-mevo-internal-target-group` — HTTP, porta 3000, health check `/health` |
| SSM Parameter | `/mevo-api/production/internal-allowed-ips` → `172.31.0.0/16` |
| Secret | `prd/mevo-api/internal-api-key` (Secrets Manager) |
| Auto Scaling | Target tracking CPU 70% — min 2, max 5 tasks |

## Recursos existentes (produção)

| Recurso | Nome |
|---|---|
| ECS Cluster | `prd-mevo-ecs-cluster` (Fargate) |
| ECS Service | `prd-mevo-api-service` — 2 tasks base, subnets privadas, sem IP público |
| Task Definition | `prd-mevo-api` — task role `prd-mevo-task-role`, execution role `prd-mevo-ecs-execution-role` |
| ECR | `production/mevo-api` (tag immutability) |
| ALB External | `prd-mevo-alb` — HTTPS:443 aberto (`0.0.0.0/0`), protegido pelo listener (allowlist `/webhook/*` + default 403) |
| Listener rule (443) | `/webhook/*` → target group (prio 10); default → fixed response 403 |
| NAT Gateway | `prd-mevo-nat-gateway` — saída para a API externa da Mevo |
| VPC Endpoints | `logs`, `ecr.api`, `ecr.dkr` (Interface) + `s3` (Gateway) |
| S3 (docs) | `prd-webdental-mevo-docs-843630347602-us-east-1-an` |
| S3 (JSON) | `prd-webdental-mevo-json-843630347602-us-east-1-an` |
| Secrets | `prd/mevo-api/database-user`, `prd/mevo-api/mevo-credentials`, `prd/mevo-api/webhook`, `prd/mevo-api/internal-api-key` |
| Log group | `/ecs/prd-mevo-api` |
| IAM User (CI/CD) | `prd-mevo-github-actions` — access key estática usada pelo GitHub Actions no deploy (candidato futuro a OIDC) |

> **Removido:** Cloudflare Access na rota interna. Substituído por ALB Internal + middleware `verifyInternalAuth`.

## Security Group do ALB External — aberto por decisão

O SG do `prd-mevo-alb` mantém ingress `0.0.0.0/0` nas portas 443 e 80. Foi uma **decisão consciente**: fechar aos ranges do Cloudflare exigiria mantê-los atualizados manualmente (os ranges mudam), e a proteção das rotas é garantida por outras camadas — o **listener** (bloqueia paths internos) e os **middlewares** (`verifyMevoWebhook` valida `CF-Connecting-IP` + Bearer; `verifyInternalAuth` valida `X-Api-Key`).

Risco residual aceito: o ALB responde a scan e tentativas diretas na internet (força bruta impraticável contra segredos de alta entropia). A alternativa sem manutenção manual é uma **customer-managed prefix list** com os ranges do Cloudflare, atualizada por Lambda agendado — não implementada.

> Sobre a porta 80: em **produção** o SG do ALB tem regra de ingress na 80 (`0.0.0.0/0`) e o listener HTTP:80 faz redirect 80→443. Em **staging** não há regra na 80. Com Cloudflare Proxy + SSL Full o Cloudflare fala 443 com a origem, então o redirect raramente é exercitado na prática — a porta 80 em produção é candidata a remoção para reduzir superfície, mas isso deve ser validado antes (confirmar que nada depende do redirect).

## Pendências de infraestrutura

1. Criar ElastiCache Valkey `app-valkey-webdental-prod` e migrar o cache dos middlewares (inconsistência entre tasks com Auto Scaling)
2. Habilitar Access Log do ALB (grava em S3) — estava desabilitado; sem ele não há auditoria retroativa de acesso
3. Lifecycle rule nos buckets S3 — alinhar retenção de 7 anos com o jurídico
4. Definir estratégia de monitoramento de percentis (p50, p95, p99)
5. Adicionar usuário `mevo_api` no banco de produção com os GRANTs necessários
