# Configuração do Ambiente Local

Este guia explica como configurar o ambiente de desenvolvimento para trabalhar com o SSO.

## Pré-requisitos

- PHP 7.0+
- Composer
- Docker (para Valkey)
- mkcert (para certificados SSL locais)
- Acesso à conta AWS AMEI (para credenciais do Cognito)

## Passo 1: Certificados SSL

O SSO usa cookies com `SameSite=None`, que exige HTTPS mesmo em desenvolvimento.

### Instalar mkcert

```bash
# Windows (Chocolatey)
choco install mkcert

# Mac
brew install mkcert

# Linux
sudo apt install mkcert
```

### Gerar certificados

```bash
# Instalar CA local
mkcert -install

# Gerar certificados para os domínios
mkcert -cert-file webdental.local.pem -key-file webdental.local-key.pem \
  webdental.local \
  "*.webdental.local" \
  localhost \
  127.0.0.1
```

### Configurar Apache/Nginx

Adicione os certificados à configuração do seu servidor local.

## Passo 2: Hosts

Adicione ao arquivo hosts:

```
# Windows: C:\Windows\System32\drivers\etc\hosts
# Linux/Mac: /etc/hosts

127.0.0.1 webdental.local
127.0.0.1 ng.webdental.local
127.0.0.1 angularjs.webdental.local
127.0.0.1 api.webdental.local
```

## Passo 3: Valkey (Redis)

### Usando Docker

```bash
docker run -d \
  --name valkey \
  -p 6379:6379 \
  valkey/valkey:latest
```

### Verificar conexão

```bash
docker exec -it valkey valkey-cli ping
# Deve retornar: PONG
```

## Passo 4: Variáveis de Ambiente

Adicione ao `.env` da API:

```env
# ============================================================
# COGNITO
# ============================================================
COGNITO_REGION=us-east-1
COGNITO_USER_POOL_ID=us-east-1_F48JuTtz8
COGNITO_CLIENT_ID=50p5fo324rsgiruhrssvf5qg9q
COGNITO_CLIENT_SECRET=seu_client_secret_aqui
COGNITO_DOMAIN=https://dev-heimdall.auth.us-east-1.amazoncognito.com
COGNITO_REDIRECT_URI=https://webdental.local/sso/callback.php
COGNITO_LOGOUT_URI=https://webdental.local/sso/auth.php
COGNITO_SCOPES=openid,email,profile

# ============================================================
# AWS CREDENTIALS (Local usa profile)
# ============================================================
AWS_COGNITO_PROFILE=cognito-secrets
AWS_COGNITO_CREDENTIALS_SECRET_ID=webdental/cognito/admin-credentials
AWS_SUPPRESS_PHP_DEPRECATION_WARNING=true

# ============================================================
# VALKEY
# ============================================================
VALKEY_HOST=127.0.0.1
VALKEY_PORT=6379
VALKEY_PASSWORD=
VALKEY_DATABASE=0

# ============================================================
# COOKIE
# ============================================================
SSO_COOKIE_DOMAIN=.webdental.local
SSO_COOKIE_SECURE=true

# ============================================================
# CORS
# ============================================================
SSO_ALLOWED_ORIGINS=https://webdental.local,https://ng.webdental.local,https://angularjs.webdental.local
```

## Passo 5: AWS CLI Profile

Configure o perfil AWS para acessar o Secrets Manager:

```bash
aws configure --profile cognito-secrets
```

Informe:
- **AWS Access Key ID**: (solicitar ao DevOps AMEI)
- **AWS Secret Access Key**: (solicitar ao DevOps AMEI)
- **Default region**: us-east-1
- **Default output format**: json

### Verificar configuração

```bash
aws secretsmanager get-secret-value \
  --secret-id webdental/cognito/admin-credentials \
  --profile cognito-secrets \
  --region us-east-1
```

## Passo 6: Instalar Dependências

```bash
cd /caminho/para/api
composer install
```

## Passo 7: Verificar Configuração

### Testar conexão com Valkey

```bash
php artisan tinker
```

```php
$redis = app('redis');
$redis->ping();
// Deve retornar: true ou "PONG"
```

### Testar conexão com Cognito

```php
$admin = app(\App\SSO\Services\Contracts\CognitoAdminServiceInterface::class);
$user = $admin->adminGetUser('seu-email@teste.com');
print_r($user);
```

## Passo 8: Testar Fluxo Completo

1. Acesse `https://webdental.local`
2. Deve redirecionar para o Cognito
3. Faça login com suas credenciais
4. Deve retornar ao Webdental autenticado
5. Verifique o cookie `webdental_session_id` no DevTools

## Troubleshooting

### Cookie não está sendo setado

- Verifique se está usando HTTPS
- Verifique se o domínio do cookie está correto (`.webdental.local`)
- Verifique se `SSO_COOKIE_SECURE=true`

### Erro de CORS

- Verifique se a origem está em `SSO_ALLOWED_ORIGINS`
- Verifique se o Angular está enviando `withCredentials: true`

### Erro de conexão com Valkey

- Verifique se o container está rodando: `docker ps`
- Verifique as variáveis `VALKEY_HOST` e `VALKEY_PORT`

### Erro de credenciais AWS

- Verifique se o profile está configurado: `aws configure list --profile cognito-secrets`
- Verifique se tem acesso ao secret: `aws secretsmanager get-secret-value ...`

### Certificado SSL inválido

- Verifique se instalou o CA do mkcert: `mkcert -install`
- Reinicie o navegador após instalar
