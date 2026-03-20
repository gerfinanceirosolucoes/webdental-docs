# Configurações (config/sso.php)

## Visão Geral

O arquivo `config/sso.php` centraliza todas as configurações do SSO. As configurações são carregadas das variáveis de ambiente.

## Arquivo Completo

```php
<?php

return [

    /*
    |--------------------------------------------------------------------------
    | AWS Cognito
    |--------------------------------------------------------------------------
    */
    
    'cognito' => [
        'region' => env('COGNITO_REGION', 'us-east-1'),
        'user_pool_id' => env('COGNITO_USER_POOL_ID'),
        'client_id' => env('COGNITO_CLIENT_ID'),
        'client_secret' => env('COGNITO_CLIENT_SECRET'),
        'domain' => env('COGNITO_DOMAIN'),
        'redirect_uri' => env('COGNITO_REDIRECT_URI'),
        'logout_uri' => env('COGNITO_LOGOUT_URI'),
        'scopes' => explode(',', env('COGNITO_SCOPES', 'openid,email,profile')),
    ],

    /*
    |--------------------------------------------------------------------------
    | AWS Credentials (para Admin API)
    |--------------------------------------------------------------------------
    */
    
    'aws' => [
        // Profile para desenvolvimento local
        'profile' => env('AWS_COGNITO_PROFILE'),
        
        // Credenciais diretas para produção
        'access_key_id' => env('AWS_COGNITO_ACCESS_KEY_ID'),
        'secret_access_key' => env('AWS_COGNITO_SECRET_ACCESS_KEY'),
        
        // Secret Manager para credenciais rotacionadas
        'credentials_secret_id' => env('AWS_COGNITO_CREDENTIALS_SECRET_ID'),
    ],

    /*
    |--------------------------------------------------------------------------
    | Valkey (Redis)
    |--------------------------------------------------------------------------
    */
    
    'valkey' => [
        'host' => env('VALKEY_HOST', '127.0.0.1'),
        'port' => env('VALKEY_PORT', 6379),
        'password' => env('VALKEY_PASSWORD'),
        'database' => env('VALKEY_DATABASE', 0),
        
        'session' => [
            'prefix' => 'webdental:session:',
            'user_prefix' => 'webdental:user_sessions:',
        ],
        
        'refresh' => [
            'prefix' => 'webdental:refresh:',
            'ttl' => 2592000, // 30 dias em segundos
        ],
    ],

    /*
    |--------------------------------------------------------------------------
    | Cookie
    |--------------------------------------------------------------------------
    */
    
    'cookie' => [
        'name' => 'webdental_session_id',
        'domain' => env('SSO_COOKIE_DOMAIN', '.webdental.local'),
        'secure' => env('SSO_COOKIE_SECURE', true),
        'http_only' => true,
        'same_site' => 'None',
        'path' => '/',
    ],

    /*
    |--------------------------------------------------------------------------
    | CORS
    |--------------------------------------------------------------------------
    */
    
    'cors' => [
        'allowed_origins' => explode(',', env('SSO_ALLOWED_ORIGINS', '')),
    ],

    /*
    |--------------------------------------------------------------------------
    | IPs Permitidos (para endpoints internos)
    |--------------------------------------------------------------------------
    */
    
    'allowed_ips' => [
        '127.0.0.1',           // Localhost
        '::1',
        // IP dos servidores que podem chamar endpoints dando bypass no SSO
        '10.0.0.0/8',          // Rede interna AWS
    ],

];
```

## Seções

### cognito

Configurações do AWS Cognito User Pool.

| Chave | Descrição | Exemplo |
|-------|-----------|---------|
| `region` | Região AWS | `us-east-1` |
| `user_pool_id` | ID do User Pool | `us-east-1_F48JuTtz8` |
| `client_id` | ID do App Client | `50p5fo324rsgiruhrssvf5qg9q` |
| `client_secret` | Secret do App Client | `xxxxx` |
| `domain` | Domínio do Cognito | `https://dev-heimdall.auth.us-east-1.amazoncognito.com` |
| `redirect_uri` | URL de callback após login | `https://webdental.local/sso/callback.php` |
| `logout_uri` | URL após logout | `https://webdental.local/sso/auth.php` |
| `scopes` | Scopes OAuth | `openid,email,profile` |

### aws

Configurações de credenciais AWS para operações administrativas.

| Chave | Descrição | Quando usar |
|-------|-----------|-------------|
| `profile` | Nome do profile AWS CLI | Desenvolvimento local |
| `access_key_id` | Access Key direta | Produção |
| `secret_access_key` | Secret Key direta | Produção |
| `credentials_secret_id` | ID do secret no Secrets Manager | Produção (credenciais rotacionadas) |

**Prioridade de carregamento:**
1. Profile (se `AWS_COGNITO_PROFILE` definido)
2. Credenciais diretas (se `AWS_COGNITO_ACCESS_KEY_ID` definido)
3. IAM Role do EC2 (fallback automático)

### valkey

Configurações do Valkey/Redis para armazenamento de sessões.

| Chave | Descrição | Default |
|-------|-----------|---------|
| `host` | Host do Valkey | `127.0.0.1` |
| `port` | Porta | `6379` |
| `password` | Senha (se houver) | `null` |
| `database` | Número do database | `0` |
| `session.prefix` | Prefixo das chaves de sessão | `webdental:session:` |
| `session.user_prefix` | Prefixo do SET de usuários | `webdental:user_sessions:` |
| `refresh.prefix` | Prefixo dos refresh tokens | `webdental:refresh:` |
| `refresh.ttl` | TTL do refresh token | `2592000` (30 dias) |

### cookie

Configurações do cookie de sessão.

| Chave | Descrição | Valor |
|-------|-----------|-------|
| `name` | Nome do cookie | `webdental_session_id` |
| `domain` | Domínio (com ponto para subdomínios) | `.webdental.local` |
| `secure` | Apenas HTTPS | `true` |
| `http_only` | Não acessível via JS | `true` |
| `same_site` | Política SameSite | `None` |
| `path` | Path do cookie | `/` |

### cors

Configurações de CORS para requisições cross-origin.

| Chave | Descrição |
|-------|-----------|
| `allowed_origins` | Lista de origens permitidas |

### allowed_ips

Lista de IPs/ranges permitidos para endpoints internos.

---

## Variáveis de Ambiente

### Desenvolvimento Local (.env)

```env
# Cognito
COGNITO_REGION=us-east-1
COGNITO_USER_POOL_ID=us-east-1_F48JuTtz8
COGNITO_CLIENT_ID=50p5fo324rsgiruhrssvf5qg9q
COGNITO_CLIENT_SECRET=seu_secret_aqui
COGNITO_DOMAIN=https://dev-heimdall.auth.us-east-1.amazoncognito.com
COGNITO_REDIRECT_URI=https://webdental.local/sso/callback.php
COGNITO_LOGOUT_URI=https://webdental.local/
COGNITO_SCOPES=openid,email,profile

# AWS (profile local)
AWS_COGNITO_PROFILE=cognito-secrets
AWS_COGNITO_CREDENTIALS_SECRET_ID=webdental/cognito/admin-credentials
AWS_SUPPRESS_PHP_DEPRECATION_WARNING=true

# Valkey
VALKEY_HOST=127.0.0.1
VALKEY_PORT=6379
VALKEY_SCHEME=tcp
VALKEY_PASSWORD=
VALKEY_DATABASE=0
VALKEY_SESSION_PREFIX=webdental:session:
VALKEY_USER_SESSION_PREFIX=webdental:user_sessions:
VALKEY_REFRESH_TTL=28800

# Cookie
SSO_COOKIE_NAME=webdental_session_id
SSO_COOKIE_DOMAIN=.webdental.local
SSO_COOKIE_SECURE=true

# CORS
SSO_ALLOWED_ORIGINS=https://webdental.local,https://ng.webdental.local,https://angularjs.webdental.local,https://api.webdental.local,https://apiwebvidas.webdental.local
```

### Produção (.env)

```env
# Cognito
COGNITO_REGION=us-east-1
COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
COGNITO_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxx
COGNITO_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxx
COGNITO_DOMAIN=https://seu-dominio.auth.us-east-1.amazoncognito.com
COGNITO_REDIRECT_URI=https://sistema.webdentalsolucoes.io/sso/callback.php
COGNITO_LOGOUT_URI=https://sistema.webdentalsolucoes.io/
COGNITO_SCOPES=openid,email,profile

# AWS (credenciais diretas injetadas pelo CI/CD)
AWS_COGNITO_ACCESS_KEY_ID=AKIAXXXXXXXXXXXXXXXXXX
AWS_COGNITO_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AWS_COGNITO_CREDENTIALS_SECRET_ID=webdental/cognito/admin-credentials

# Valkey
VALKEY_HOST=valkey.webdental.internal
VALKEY_PORT=6379
VALKEY_SCHEME=tls
VALKEY_PASSWORD=sua_senha_segura
VALKEY_DATABASE=0
VALKEY_SESSION_PREFIX=webdental:session:
VALKEY_USER_SESSION_PREFIX=webdental:user_sessions:
VALKEY_REFRESH_TTL=28800

# Cookie
SSO_COOKIE_NAME=webdental_session_id
SSO_COOKIE_DOMAIN=.sistema.webdentalsolucoes.io
SSO_COOKIE_SECURE=true

# CORS
SSO_ALLOWED_ORIGINS=https://sistema.webdentalsolucoes.io,https://ng.webdentalsolucoes.io.br,https://novo.webdentalsolucoes.io,https://api.webdentalsolucoes.io,https://apiwebvidas.webdentalsolucoes.io
```

## Acessando Configurações no Código

```php
// Via helper
$userPoolId = config('sso.cognito.user_pool_id');
$cookieDomain = config('sso.cookie.domain');

// Via facade
$valkeyHost = Config::get('sso.valkey.host');

// Array completo
$cognitoConfig = config('sso.cognito');
```
