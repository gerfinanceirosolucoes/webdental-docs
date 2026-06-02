# Arquitetura do Projeto Mevo API

## Contexto de Negócio

A Mevo API é um serviço intermediário que integra o Webdental com a plataforma **Mevo** para geração de receituários digitais, exames e atestados odontológicos. O serviço é invocado pelo backend Laravel do Webdental e também recebe notificações via webhook da empresa Mevo.

---

## Diagrama de Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                        Webdental                            │
│                                                             │
│   Angular (browser)  →  Laravel/PHP (SRV01)                │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTPS (rede interna VPC)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    AWS (sa-east-1 — Staging)                │
│                                                             │
│   Cloudflare DNS                                            │
│        │                                                    │
│        ▼                                                    │
│   Application Load Balancer (HTTPS:443)                     │
│        │                                                    │
│        ▼                                                    │
│   ECS Fargate — Mevo API (Node.js/TypeScript)               │
│        │                    │                               │
│        ▼                    ▼                               │
│   MariaDB (BD Staging)   S3 Bucket (documentos)            │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ VPC Endpoints (acesso privado sem internet)         │  │
│   │  • logs (CloudWatch)                                │  │
│   │  • ecr.api + ecr.dkr (pull de imagens)              │  │
│   │  • s3                                               │  │
│   └─────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTPS (internet)
                               ▼
                    API Externa da Mevo
```

---

## Fluxos Principais

### Fluxo 1 — Geração de Receituário

```
1. Dentista preenche formulário no Webdental (Angular)
2. Angular chama o backend Laravel (PHP)
3. Laravel valida a autenticação do dentista
4. Laravel chama a Mevo API (rede interna da VPC)
5. Mevo API autentica na API externa da Mevo
6. Mevo API chama a API da Mevo para gerar o receituário
7. Mevo API salva o documento no S3 e os metadados no MariaDB
8. Mevo API retorna a URL pré-assinada para o Laravel
9. Laravel retorna para o Angular
10. Dentista vê o receituário na tela
```

### Fluxo 2 — Webhook

```
1. Mevo (empresa externa) chama POST /webhook/receita
2. ALB verifica se o IP está na whitelist (Security Group)
3. Middleware valida o token no header X-Mevo-Token
4. Mevo API processa o payload e registra no banco
5. Mevo API retorna 200 OK
```

### Fluxo 3 — Visualização de Documento

```
1. Webdental solicita URL do documento (GET /documento/:id/url)
2. Mevo API busca a chave S3 no banco de dados
3. Mevo API gera uma URL pré-assinada com expiração de 15 minutos
4. Mevo API retorna a URL para o Webdental
5. Browser do dentista acessa o S3 diretamente via URL pré-assinada
```

---

## Decisões Arquiteturais

| Decisão | Escolha | Justificativa |
|---|---|---|
| Compute | ECS Fargate | Projeto estruturado em Express/Node, sem refatoração para Lambda, custo ~$9/mês |
| Storage de documentos | Amazon S3 | Padrão de mercado para binários, versionamento LGPD, custo mínimo |
| JSON de retorno da Mevo | S3 + referência no banco | JSON grande não deve ficar em banco relacional |
| Dados analíticos | MariaDB BD Staging | Campos indexáveis para relatórios futuros |
| Credenciais em Fargate | IAM Role na Task Definition | Sem Access Keys no código — padrão de segurança AWS |
| Autenticação webhook | Token fixo + Whitelist de IP | Limitação atual da Mevo; evoluir para HMAC no futuro |
| Cluster ECS | 1 cluster por projeto | Isolamento, autonomia de deploy, preparação para IaC |
| CI/CD | GitHub Actions | Consistência com demais projetos da empresa |

---

## Stack Tecnológica

| Camada | Tecnologia |
|---|---|
| Runtime | Node.js 20 (Alpine) |
| Linguagem | TypeScript (strict) |
| Framework | Express 5 |
| ORM | TypeORM |
| Validação | Zod v4 |
| HTTP Client | Axios |
| Storage SDK | AWS SDK v3 (@aws-sdk/client-s3) |
| Containerização | Docker (multi-stage build) |
| Orquestração | AWS ECS Fargate |
| Registry | Amazon ECR |
| Load Balancer | AWS Application Load Balancer |
| Banco de dados | MariaDB (instância compartilhada) |
| Storage | Amazon S3 |
| Secrets | AWS Secrets Manager |
| Logs | Amazon CloudWatch Logs |
| CI/CD | GitHub Actions |
