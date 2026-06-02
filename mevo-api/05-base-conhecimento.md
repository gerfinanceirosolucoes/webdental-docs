# Base de Conhecimento — Tecnologias Utilizadas

## 1. AWS ECS Fargate

### O que é

O **Elastic Container Service (ECS)** é o serviço da AWS para executar containers Docker. O **Fargate** é o modo de execução onde a AWS gerencia toda a infraestrutura de servidores — você apenas define o container e os recursos (CPU/memória) que ele precisa.

**Analogia:** é como alugar um apartamento mobiliado. Você não precisa se preocupar com a construção, encanamento ou fiação — só usa o espaço.

### Conceitos Principais

| Conceito | O que é | Analogia |
|---|---|---|
| **Task Definition** | Receita do container — imagem, recursos, variáveis de ambiente | Planta de um apartamento |
| **Task** | Uma instância rodando da Task Definition | O apartamento em si |
| **Service** | Garante que N tasks estejam sempre rodando | Síndico que garante N apartamentos ocupados |
| **Cluster** | Agrupamento lógico de serviços | Condomínio |

### Como o deploy funciona

```
Novo código commitado
        │
        ▼
GitHub Actions faz push de nova imagem no ECR
        │
        ▼
GitHub Actions registra nova Task Definition (aponta para nova imagem)
        │
        ▼
ECS Service inicia nova task com a nova imagem
        │
        ▼
ECS aguarda nova task ficar saudável (health check)
        │
        ▼
ECS para a task antiga (rolling update)
```

### IAM Roles no ECS

O ECS usa duas roles distintas — muita gente confunde:

| Role | Quem usa | Para quê |
|---|---|---|
| **Task Role** (`hmg-mevo-task-role`) | O seu código dentro do container | Acessar S3, banco, outros serviços AWS |
| **Execution Role** (`hmg-mevo-ecs-execution-role`) | O agente ECS (antes do container subir) | Fazer pull da imagem no ECR, criar logs no CloudWatch, buscar secrets no Secrets Manager |

---

## 2. Application Load Balancer (ALB)

### O que é

O ALB é um ponto de entrada único e estável para a sua aplicação. Ele fica na frente dos containers e distribui o tráfego para as tasks saudáveis.

### Por que precisamos

Sem o ALB, o problema seria:
- Cada task Fargate tem um IP privado que **muda a cada deploy**
- Não há como expor HTTPS diretamente no container
- Sem health check automático de roteamento

Com o ALB:
```
stg-mevo-api.webdentalsolucoes.io (IP fixo via DNS)
        │
        ▼
ALB (gerencia certificado HTTPS)
        │
        ▼ roteia apenas para tasks saudáveis
Task Fargate (172.31.x.x:3000) ← IP muda a cada deploy, mas o ALB se atualiza automaticamente
```

### Componentes

| Componente | O que é |
|---|---|
| **Listener** | Porta onde o ALB escuta (443 HTTPS, 80 HTTP) |
| **Target Group** | Lista de destinos saudáveis (tasks Fargate) |
| **Health Check** | O ALB chama `/health` a cada 30s para verificar se a task está OK |
| **Security Group** | Define quem pode acessar o ALB (whitelist de IPs) |

### Terminação TLS

O certificado SSL fica no ALB — não no container. O ALB recebe HTTPS do cliente e encaminha HTTP para o container internamente. Isso simplifica muito a configuração da aplicação.

```
Cliente → HTTPS → ALB → HTTP:3000 → Container
```

---

## 3. Amazon S3 e Presigned URLs

### O que é o S3

O **Simple Storage Service (S3)** é um serviço de armazenamento de objetos (arquivos). É onde armazenamos os PDFs de receitas, exames e atestados.

### Estrutura de chaves

No S3, os arquivos são organizados por "chaves" (paths):

```
{tipo}/{cd_paciente}/{timestamp}_{uuid}_{arquivo}.pdf

Exemplos:
receitas/L00700020190318091651/20260521143022_a6cc44a2_REC-001.pdf
exames/L00700020190318091651/20260521160000_b7dd55e3_EXA-001.pdf
```

### O que salvar no banco

**Nunca** salvar a URL completa do S3 no banco — ela expira e muda. Salvar apenas a **chave S3** (o path):

```typescript
// ❌ Errado — URL expira
db.save({ url: 'https://bucket.s3.amazonaws.com/receitas/...' });

// ✅ Correto — chave permanente
db.save({ s3_key: 'receitas/L007.../20260521_REC-001.pdf' });
```

### Presigned URLs

Para dar acesso temporário a um arquivo privado, geramos uma **URL pré-assinada**:

```typescript
import { GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const url = await getSignedUrl(s3Client, new GetObjectCommand({
  Bucket: process.env.AWS_BUCKET,
  Key: 'receitas/L007.../REC-001.pdf',
}), { expiresIn: 900 }); // 15 minutos
```

A URL gerada funciona por 15 minutos — qualquer pessoa com o link pode baixar o arquivo nesse período. Após expirar, a URL para de funcionar automaticamente.

**Por que é seguro:** o bucket é 100% privado. Sem a URL pré-assinada, ninguém consegue acessar o arquivo.

---

## 4. AWS Secrets Manager

### O que é

O Secrets Manager armazena credenciais sensíveis (senhas, tokens, chaves de API) de forma criptografada. A aplicação busca os valores em runtime — nunca ficam hardcoded no código ou nas variáveis de ambiente em texto simples.

### Como funciona no ECS

```
Task Definition define:
{
  "name": "DB_PASSWORD",
  "valueFrom": "arn:aws:secretsmanager:...:hmg/mevo-api/database-user:DB_PASSWORD::"
}
        │
        ▼ (antes do container subir)
ECS Execution Role busca o valor no Secrets Manager
        │
        ▼
Container inicia com DB_PASSWORD já disponível como variável de ambiente
```

O código Node.js acessa normalmente via `process.env.DB_PASSWORD` — sem saber de onde veio.

### Convenção de nomenclatura

```
{ambiente}/mevo-api/{tipo}

Exemplos:
hmg/mevo-api/database-user     ← credenciais do banco
hmg/mevo-api/mevo-credentials  ← credenciais da API Mevo
hmg/mevo-api/webhook           ← token do webhook
prd/mevo-api/database-user     ← produção
```

---

## 5. VPC Endpoints

### O problema que resolvem

Por padrão, quando um container no ECS precisa acessar serviços AWS (CloudWatch, ECR, S3), o tráfego sai pela internet pública e retorna — mesmo sendo serviços da mesma AWS.

Isso é lento, tem custo de transferência e exige que o container tenha IP público.

### A solução

Os **VPC Endpoints** criam um "atalho" privado dentro da VPC para acessar serviços AWS sem sair para a internet:

```
Sem VPC Endpoint:
Container → Internet pública → CloudWatch

Com VPC Endpoint:
Container → VPC Endpoint (rede privada) → CloudWatch
```

### Endpoints configurados no projeto

| Serviço | Para quê |
|---|---|
| `logs` (CloudWatch) | Container envia logs |
| `ecr.api` + `ecr.dkr` | ECS faz pull da imagem Docker |
| `s3` | Container acessa o bucket S3 |

### Ponto de atenção

Os VPC Endpoints têm **Security Groups** que controlam quem pode acessá-los. Ao criar um novo serviço ECS, sempre:

1. Associar o novo Security Group aos VPC Endpoints
2. Adicionar regra ingress 443 no Security Group dos endpoints para o novo SG

---

## 6. GitHub Actions com AWS

### O que é

O **GitHub Actions** é o serviço de CI/CD integrado ao GitHub. Permite automatizar tarefas (build, test, deploy) a cada evento no repositório (push, PR, tag).

### Fluxo de autenticação com AWS

O pipeline usa credenciais de um IAM User dedicado (`hmg-mevo-github-actions`) armazenadas como secrets no GitHub:

```
GitHub Actions
    │ usa secrets: AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY
    ▼
aws-actions/configure-aws-credentials  ← action oficial da AWS
    │ configura credenciais temporárias
    ▼
Próximos steps autenticados na AWS
```

### Actions utilizadas

| Action | Versão | O que faz |
|---|---|---|
| `actions/checkout` | v4 | Baixa o código do repositório |
| `aws-actions/configure-aws-credentials` | v4 | Configura credenciais AWS |
| `aws-actions/amazon-ecr-login` | v2 | Autentica no ECR para push/pull de imagens |
| `aws-actions/amazon-ecs-render-task-definition` | v1 | Atualiza a Task Definition com a nova imagem |
| `aws-actions/amazon-ecs-deploy-task-definition` | v2 | Faz deploy no ECS e aguarda estabilização |

### Environments

O projeto usa **GitHub Environments** para separar configurações por ambiente:

```
GitHub → repositório → Settings → Environments → staging
```

Vantagens:
- Secrets e variables específicos por ambiente
- Possibilidade de adicionar proteções (aprovação manual em produção)
- Auditoria separada por ambiente

### Troubleshooting comum

**`Unexpected key 'enableFaultInjection' found in params`**
Usar `amazon-ecs-deploy-task-definition@v2` (não v1).

**`AccessDenied` ao registrar Task Definition**
Verificar se a policy `Hmg-MevoGitHubActionsPolicy` tem as permissões `ecs:RegisterTaskDefinition` e `iam:PassRole`.
