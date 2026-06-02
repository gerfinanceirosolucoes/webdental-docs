# Guia de Deploy — Mevo API

## Visão Geral

| Ambiente | Gatilho | Responsável | Aprovação |
|---|---|---|---|
| Staging | Push/merge na branch `staging` | Automático (GitHub Actions) | Não requerida |
| Produção | Push de tag `v1.0.0` | TechLead | Manual |

---

## Deploy em Staging

### Fluxo Automático

```
1. Desenvolva na sua branch feat/*
2. Abra PR para staging
3. Após aprovação, faça merge
4. GitHub Actions dispara automaticamente
5. Pipeline faz build, push ECR e deploy ECS
6. Aguarde ~3 minutos para o deploy completar
7. Valide em https://stg-mevo-api.webdentalsolucoes.io/health
```

### Acompanhar o Pipeline

```
GitHub → repositório mevo_api → Actions → Deploy Staging
```

O pipeline tem 5 etapas:

| Etapa | O que faz | Tempo estimado |
|---|---|---|
| Checkout | Baixa o código | ~10s |
| Configure AWS credentials | Autentica na AWS | ~5s |
| Login to Amazon ECR | Autentica no registry | ~5s |
| Build, tag and push image | Build Docker + push ECR | ~2-3 min |
| Deploy to ECS | Atualiza Task Definition + aguarda estabilização | ~3-5 min |

### Tag da Imagem

Cada deploy em staging gera uma tag no formato:

```
hmg-{git-sha}
```

Exemplo: `hmg-2008da8c158c6af0eecff72971783fe0235b5891`

Isso garante rastreabilidade total — você consegue identificar exatamente qual commit está rodando em staging.

---

## Deploy em Produção

### Pré-requisitos

- [ ] Código validado e estável em staging
- [ ] Testes realizados no ambiente de staging
- [ ] Aprovação do TechLead

### Como Fazer

```bash
# 1. Certifique-se de estar na branch staging atualizada
git checkout staging
git pull origin staging

# 2. Crie a tag de versão (seguir semver)
git tag v1.0.0

# 3. Faça o push da tag
git push origin v1.0.0
```

O GitHub Actions dispara automaticamente ao detectar a tag `v1.0.0`.

### Convenção de Versionamento (Semver)

| Tipo de mudança | Exemplo | Quando usar |
|---|---|---|
| Patch (bugfix) | `v1.0.1` | Correção de bug sem nova funcionalidade |
| Minor (feature) | `v1.1.0` | Nova funcionalidade compatível |
| Major (breaking) | `v2.0.0` | Mudança incompatível com versão anterior |

---

## Rollback

### Staging

Em caso de problema após o deploy, force um novo deploy com a Task Definition anterior:

```bash
# Listar revisões disponíveis
aws ecs list-task-definitions \
  --family-prefix hmg-mevo-api \
  --region sa-east-1

# Fazer deploy de uma revisão específica
aws ecs update-service \
  --cluster hmg-mevo-ecs-cluster \
  --service hmg-mevo-api-service \
  --task-definition hmg-mevo-api:NUMERO_DA_REVISAO \
  --region sa-east-1
```

### Produção

Mesmo processo, substituindo os nomes dos recursos de produção.

---

## Verificando o Status do Deploy

### Via Console AWS

```
ECS → hmg-mevo-ecs-cluster → hmg-mevo-api-service → aba Deployments
```

O deploy está completo quando:
- Status: `PRIMARY`
- Running count = Desired count (1/1)
- Deployment status: `COMPLETED`

### Via Curl

```bash
curl https://stg-mevo-api.webdentalsolucoes.io/health
# Resposta esperada: {"status":"ok"}
```

### Via CloudWatch

```
CloudWatch → Log groups → /ecs/hmg-mevo-api → log stream mais recente
```

Logs esperados após deploy bem-sucedido:
```
Database connected
API rodando 3000
```

---

## Problemas Comuns

### Task não sobe — CloudWatch inacessível

**Sintoma:** `ResourceInitializationError: failed to validate logger args`

**Causa:** Security Group da task não está associado aos VPC Endpoints ou não tem ingress 443 liberado.

**Solução:** verificar se `sg-063937277b90624d9` está associado aos VPC Endpoints e se o `sg-0a7a137decbbf69ec` tem regra ingress 443 para o SG da task. Ver seção de Lições Aprendidas no Runbook de Infraestrutura.

---

### Task não conecta no banco

**Sintoma:** `Database connection failed: ECONNREFUSED`

**Causa:** variável `DB_HOST` não configurada ou Security Group do banco não libera a porta 3306 para a task.

**Solução:** verificar variáveis de ambiente na Task Definition e regras de ingress no Security Group do banco (`sg-085bd8d87a78e0807`).

---

### Pipeline falha no build TypeScript

**Sintoma:** `error TS2345` ou similar no step "Build, tag and push image"

**Causa:** código com erros de tipagem TypeScript.

**Solução:** rodar `npm run validate` localmente, corrigir os erros e fazer novo push.
