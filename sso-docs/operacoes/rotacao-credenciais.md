# Rotação de Credenciais

## Visão Geral

O SSO utiliza um sistema de rotação automática de credenciais AWS para aumentar a segurança. As credenciais usadas para acessar o Cognito são rotacionadas a cada 15 dias.

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FLUXO DE CREDENCIAIS                           │
│                                                                             │
│   ┌─────────────┐     ┌─────────────────────┐     ┌─────────────────────┐  │
│   │             │     │                     │     │                     │  │
│   │  Laravel    │────►│   Secrets Manager   │────►│      Cognito        │  │
│   │    API      │     │   (Conta AMEI)      │     │   (Conta AMEI)      │  │
│   │             │     │                     │     │                     │  │
│   └─────────────┘     └─────────────────────┘     └─────────────────────┘  │
│         │                      │                                            │
│         │                      │                                            │
│         │              ┌───────┴───────┐                                   │
│         │              │               │                                   │
│         │              ▼               ▼                                   │
│         │      ┌─────────────┐ ┌─────────────┐                             │
│         │      │  Current    │ │  Pending    │                             │
│         │      │  Secret     │ │  Secret     │                             │
│         │      │  (ativa)    │ │  (próxima)  │                             │
│         │      └─────────────┘ └─────────────┘                             │
│         │                                                                   │
│   ┌─────┴─────────────────────────────────────────────────────────────┐   │
│   │                                                                    │   │
│   │   Credenciais fixas (.env)                                        │   │
│   │   AWS_COGNITO_ACCESS_KEY_ID=AKIA...                               │   │
│   │   AWS_COGNITO_SECRET_ACCESS_KEY=...                               │   │
│   │                                                                    │   │
│   │   Estas credenciais APENAS acessam o Secrets Manager              │   │
│   │   para buscar as credenciais rotacionadas                         │   │
│   │                                                                    │   │
│   └────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Como Funciona

### 1. Credenciais Fixas

Armazenadas no `.env` (injetadas via CI/CD):
- `AWS_COGNITO_ACCESS_KEY_ID`
- `AWS_COGNITO_SECRET_ACCESS_KEY`

Estas credenciais têm **apenas** permissão para:
- `secretsmanager:GetSecretValue`

### 2. Credenciais Rotacionadas

Armazenadas no AWS Secrets Manager:
- Secret ID: `webdental/cognito/admin-credentials`
- Rotação: A cada 15 dias

Estas credenciais têm permissões para:
- `cognito-idp:Admin*`
- `cognito-idp:GlobalSignOut`
- `cognito-idp:ListUsers`

### 3. Fluxo de Uso

```php
// AwsCredentialsService.php

public function getCognitoAdminCredentials(): array
{
    // 1. Usa credenciais fixas para acessar Secrets Manager
    $secretsClient = new SecretsManagerClient([
        'region' => 'us-east-1',
        'credentials' => [
            'key' => env('AWS_COGNITO_ACCESS_KEY_ID'),
            'secret' => env('AWS_COGNITO_SECRET_ACCESS_KEY'),
        ],
    ]);

    // 2. Busca credenciais rotacionadas
    $result = $secretsClient->getSecretValue([
        'SecretId' => env('AWS_COGNITO_CREDENTIALS_SECRET_ID'),
    ]);

    // 3. Retorna credenciais para uso no CognitoAdminService
    return json_decode($result['SecretString'], true);
}
```

---

## Configuração no AWS

### Secret no Secrets Manager

**Nome:** `webdental/cognito/admin-credentials`

**Conteúdo:**
```json
{
    "accessKeyId": "AKIAXXXXXXXXXXXXXXXX",
    "secretAccessKey": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

### Rotação Automática

1. Acesse o Secrets Manager no console AWS
2. Selecione o secret
3. Clique em "Rotation"
4. Configure:
   - **Rotation interval:** 15 days
   - **Lambda function:** (criar ou usar existente)

### Lambda de Rotação

A Lambda de rotação deve:
1. Criar novo par de credenciais IAM
2. Atualizar o secret com as novas credenciais
3. Testar as novas credenciais
4. Remover as credenciais antigas

---

## Políticas IAM

### IAM User para Secrets Manager (fixo)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue"
            ],
            "Resource": "arn:aws:secretsmanager:us-east-1:237586137945:secret:webdental/cognito/admin-credentials-*"
        }
    ]
}
```

### IAM User para Cognito (rotacionado)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "cognito-idp:AdminGetUser",
                "cognito-idp:AdminCreateUser",
                "cognito-idp:AdminDisableUser",
                "cognito-idp:AdminEnableUser",
                "cognito-idp:AdminUpdateUserAttributes",
                "cognito-idp:AdminDeleteUser",
                "cognito-idp:AdminSetUserPassword",
                "cognito-idp:AdminUserGlobalSignOut",
                "cognito-idp:GlobalSignOut",
                "cognito-idp:ListUsers"
            ],
            "Resource": "arn:aws:cognito-idp:us-east-1:237586137945:userpool/us-east-1_F48JuTtz8"
        }
    ]
}
```

---

## Troubleshooting

### Erro: Unable to retrieve credentials

**Causa:** Credenciais fixas inválidas ou sem permissão.

**Solução:**
1. Verificar `AWS_COGNITO_ACCESS_KEY_ID` e `AWS_COGNITO_SECRET_ACCESS_KEY` no `.env`
2. Verificar se o IAM User tem permissão para `secretsmanager:GetSecretValue`

### Erro: Access Denied ao chamar Cognito

**Causa:** Credenciais rotacionadas expiraram ou foram deletadas.

**Solução:**
1. Verificar se o secret existe no Secrets Manager
2. Verificar se a rotação está funcionando
3. Verificar se o IAM User do secret tem permissões no Cognito

### Erro: Secret not found

**Causa:** Secret ID incorreto ou secret deletado.

**Solução:**
1. Verificar `AWS_COGNITO_CREDENTIALS_SECRET_ID` no `.env`
2. Verificar se o secret existe no Secrets Manager

---

## Monitoramento

### Verificar última rotação

```bash
aws secretsmanager describe-secret \
    --secret-id webdental/cognito/admin-credentials \
    --query 'LastRotatedDate'
```

### Verificar próxima rotação

```bash
aws secretsmanager describe-secret \
    --secret-id webdental/cognito/admin-credentials \
    --query 'NextRotationDate'
```

### Testar credenciais manualmente

```bash
# Buscar secret
aws secretsmanager get-secret-value \
    --secret-id webdental/cognito/admin-credentials \
    --query 'SecretString' \
    --output text

# Usar credenciais para testar Cognito
export AWS_ACCESS_KEY_ID=<do secret>
export AWS_SECRET_ACCESS_KEY=<do secret>

aws cognito-idp admin-get-user \
    --user-pool-id us-east-1_F48JuTtz8 \
    --username email@teste.com
```

---

## Rotação Manual (Emergência)

Se precisar rotacionar manualmente:

```bash
# 1. Criar novo par de credenciais
aws iam create-access-key --user-name cognito-admin-user

# 2. Atualizar o secret
aws secretsmanager put-secret-value \
    --secret-id webdental/cognito/admin-credentials \
    --secret-string '{"accessKeyId":"AKIA...","secretAccessKey":"..."}'

# 3. Testar
aws cognito-idp admin-get-user \
    --user-pool-id us-east-1_F48JuTtz8 \
    --username email@teste.com

# 4. Deletar credenciais antigas
aws iam delete-access-key \
    --user-name cognito-admin-user \
    --access-key-id AKIA_OLD...
```

---

## Segurança

### Boas práticas

- ✅ Credenciais fixas têm permissão mínima (apenas Secrets Manager)
- ✅ Credenciais rotacionadas têm escopo limitado ao User Pool específico
- ✅ Rotação automática a cada 15 dias
- ✅ Credenciais nunca commitadas no Git
- ✅ Injeção via CI/CD (GitHub Secrets)

### O que não fazer

- ❌ Commitar credenciais no repositório
- ❌ Compartilhar credenciais por email/chat
- ❌ Usar as mesmas credenciais em dev/staging/prod
- ❌ Dar permissões além do necessário
