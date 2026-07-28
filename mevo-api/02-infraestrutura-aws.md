# Infraestrutura

## Recursos existentes (produção)

| Recurso | Nome |
|---|---|
| ECS Cluster | `prd-mevo-ecs-cluster` (Fargate) |
| ECS Service | `prd-mevo-api-service` — 2 tasks base, subnets privadas, sem IP público |
| Task Definition | `prd-mevo-api` — 1 vCPU / 2 GB; task role `prd-mevo-task-role`, execution role `prd-mevo-ecs-execution-role` |
| ECR | `production/mevo-api` (tag immutability) |
| ALB External | `prd-mevo-alb` — HTTPS:443 aberto (`0.0.0.0/0`), protegido pelo listener (allowlist `/webhook/*` + default 403) |
| Listener rule (443) | `/webhook/*` → target group (priority 10); default → fixed response 403 |
| ALB Internal | `prd-mevo-alb-internal` — HTTP:80, alcançável só de dentro da VPC |
| Target Groups | `prd-mevo-target-group` (externo) e `prd-mevo-internal-target-group` (interno) — IP, HTTP:3000, health check `/health` |
| NAT Gateway + EIP | `prd-mevo-nat-gateway` — saída para a API externa da Mevo; **IP de saída fixo `52.2.134.132`** (o IP que a Mevo vê; informar se pedir whitelist de origem) |
| VPC Endpoints | `logs`, `ecr.api`, `ecr.dkr` (Interface) + `s3` (Gateway) |
| Security Groups | ALB (`prd-mevo-alb-sg`) · ALB Internal (`prd-mevo-alb-internal-sg`) · Service (`prd-mevo-ecs-service-sg`, só dos ALBs) · VPC Endpoints (`prd-mevo-endpoints-sg`, só do Service) |
| S3 | docs: `prd-webdental-mevo-docs-...` · JSON: `prd-webdental-mevo-json-...` |
| Secrets | `prd/mevo-api/database-user`, `prd/mevo-api/mevo-credentials`, `prd/mevo-api/webhook`, `prd/mevo-api/internal-api-key` |
| SSM Parameters | `/mevo-api/production/internal-allowed-ips` (CIDR interno) · `/mevo-api/production/webhook-ips` (IPs da Mevo) |
| Certificado ACM | `mevo-api.webdentalsolucoes.io` (mesma região do ALB; validação DNS) |
| DNS (Cloudflare) | `mevo-api.webdentalsolucoes.io` → ALB External (CNAME, Proxied) |
| Log group | `/ecs/prd-mevo-api` (retenção 30 dias) |
| IAM User (CI/CD) | `prd-mevo-github-actions` — access key estática usada pelo GitHub Actions no deploy |
