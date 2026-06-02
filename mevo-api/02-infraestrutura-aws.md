# Infraestrutura AWS — Mevo API

## Inventário de Recursos

### Staging (sa-east-1)

| Recurso | Nome | Observação |
|---|---|---|
| S3 Bucket | `stg-webdental-mevo-docs-843630347602-sa-east-1-an` | Versionamento habilitado, acesso público bloqueado |
| IAM User (dev local) | `stg-mevo-s3-user` | Access Keys para desenvolvimento local |
| IAM Policy (dev local) | `Stg-MevoServiceAccountS3Policy` | Acesso apenas ao bucket de staging |
| IAM User (CI/CD) | `hmg-mevo-github-actions` | Usado pelo GitHub Actions para deploy |
| IAM Policy (CI/CD) | `Hmg-MevoGitHubActionsPolicy` | Permissões mínimas para ECR + ECS |
| IAM Task Role | `hmg-mevo-task-role` | Assumida pelo container — acesso ao S3 |
| IAM Execution Role | `hmg-mevo-ecs-execution-role` | Usada pelo agente ECS — ECR, CloudWatch, Secrets |
| ECR Repository | `staging/mevo-api` | Tag immutability habilitado |
| ECS Cluster | `hmg-mevo-ecs-cluster` | 1 cluster por projeto |
| Task Definition | `hmg-mevo-api` | 0.5 vCPU / 1 GB RAM |
| ECS Service | `hmg-mevo-api-service` | 1 task, vinculado ao ALB |
| ALB | `hmg-mevo-alb` | HTTPS:443 + redirect HTTP:80 |
| Target Group | `hmg-mevo-target-group` | IP type, porta 3000, health check /health |
| ACM Certificate | `stg-mevo-api.webdentalsolucoes.io` | Validado via DNS (Cloudflare) |
| CloudWatch Log Group | `/ecs/hmg-mevo-api` | Retenção: 1 mês |
| Secrets Manager | `hmg/mevo-api/database-user` | DB_USERNAME + DB_PASSWORD |
| Secrets Manager | `hmg/mevo-api/mevo-credentials` | MEVO_USERNAME + MEVO_PASSWORD |
| Secrets Manager | `hmg/mevo-api/webhook` | MEVO_WEBHOOK_TOKEN |

### Produção (us-east-1)

> A criar — seguir o runbook de infraestrutura substituindo os valores conforme tabela de diferenças.

---

## Configuração de Rede

### VPC

| Ambiente | VPC ID | CIDR |
|---|---|---|
| Staging | `vpc-58a7ec3c` (default_aws) | `172.31.0.0/16` |
| Produção | `vpc-81f5f4f8` | `172.31.0.0/16` |

### Subnets de Staging

| Subnet ID | AZ | CIDR |
|---|---|---|
| `subnet-74c6223d` | sa-east-1b | `172.31.32.0/20` |
| `subnet-3ad81a5d` | sa-east-1a | `172.31.0.0/20` |
| `subnet-787bd920` | sa-east-1c | `172.31.16.0/20` |

### Security Groups

| Nome | ID | Propósito |
|---|---|---|
| `hmg-mevo-alb-sg` | — | ALB — ingress 443/80 whitelist, egress 3000 para task |
| `hmg-mevo-service-sg` | `sg-063937277b90624d9` | Task ECS — ingress 3000 da VPC, egress all |
| `hmg-mensageria-ecs-security-group` | `sg-0a7a137decbbf69ec` | SG dos VPC Endpoints — deve incluir ingress 443 para hmg-mevo-service-sg |

> **Atenção:** ao criar novos serviços ECS que precisam de CloudWatch/ECR via VPC Endpoints, sempre adicionar o novo Security Group ao `sg-0a7a137decbbf69ec` (ingress 443) e associar o novo SG aos VPC Endpoints.

### VPC Endpoints (Compartilhados)

| Endpoint | Serviço | ID |
|---|---|---|
| CloudWatch Logs | `com.amazonaws.sa-east-1.logs` | `vpce-0e3ebe6d5b4cfb964` |
| ECR API | `com.amazonaws.sa-east-1.ecr.api` | `vpce-0486928ea24fc59e4` |
| ECR Docker | `com.amazonaws.sa-east-1.ecr.dkr` | `vpce-06cbe9c8e0210e8ce` |
| S3 | `com.amazonaws.sa-east-1.s3` | `vpce-0e9b7b89587624565` |

---

## Variáveis de Ambiente

### Configuração (Task Definition)

| Variável | Valor em Staging | Observação |
|---|---|---|
| `NODE_ENV` | `staging` | |
| `AWS_REGION` | `sa-east-1` | |
| `S3_BUCKET` | `stg-webdental-mevo-docs-843630347602-sa-east-1-an` | |
| `S3_PREFIX` | `staging` | Vazio em produção |
| `DB_HOST` | IP do banco de staging | |
| `DB_PORT` | `3306` | |
| `DB_DATABASE` | Nome do banco | |
| `MEVO_API_URL` | URL da API da Mevo | |

### Secrets (AWS Secrets Manager)

| Variável | Secret | Chave |
|---|---|---|
| `DB_USERNAME` | `hmg/mevo-api/database-user` | `DB_USERNAME` |
| `DB_PASSWORD` | `hmg/mevo-api/database-user` | `DB_PASSWORD` |
| `MEVO_USERNAME` | `hmg/mevo-api/mevo-credentials` | `MEVO_USERNAME` |
| `MEVO_PASSWORD` | `hmg/mevo-api/mevo-credentials` | `MEVO_PASSWORD` |
| `MEVO_WEBHOOK_TOKEN` | `hmg/mevo-api/webhook` | `MEVO_WEBHOOK_TOKEN` |

---

## Endpoints

| Ambiente | URL Base |
|---|---|
| Staging | `https://stg-mevo-api.webdentalsolucoes.io` |
| Produção | A definir |
