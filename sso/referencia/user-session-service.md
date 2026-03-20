# UserSessionService

## O que é

O `UserSessionService` é o serviço central de gerenciamento de sessões do SSO. Ele orquestra a criação, validação, renovação e invalidação de sessões, integrando Valkey, Cognito e o banco de dados local.

## Responsabilidades

| Responsabilidade | Descrição |
|------------------|-----------|
| Criar sessões | Após login, cria sessão no Valkey com dados do usuário |
| Validar sessões | Verifica se sessão existe e não expirou |
| Renovar sessões | Usa refresh token para obter novos tokens |
| Invalidar sessões | Remove sessão no logout |
| Logout global | Invalida todas as sessões de um usuário |

## Como usar

### Injeção de Dependência

```php
use App\SSO\Services\Contracts\UserSessionServiceInterface;

class AuthController extends Controller
{
    private $sessionService;

    public function __construct(UserSessionServiceInterface $sessionService)
    {
        $this->sessionService = $sessionService;
    }
}
```

### Via Container

```php
$sessionService = app(UserSessionServiceInterface::class);
```

## Referência de Métodos

### createSession

Cria uma nova sessão após autenticação bem-sucedida.

```php
$sessionData = $sessionService->createSession(
    $idToken,
    $accessToken,
    $refreshToken,
    $expiresIn,
    $cognitoSub
);
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$idToken` | string | ID Token do Cognito |
| `$accessToken` | string | Access Token do Cognito |
| `$refreshToken` | string | Refresh Token do Cognito |
| `$expiresIn` | int | Tempo de expiração em segundos |
| `$cognitoSub` | string | Sub (UUID) do usuário no Cognito |

**Retorno:**
```php
[
    'session_id' => 'V1StGXR8_Z5jdHi6B-myT',  // NanoID 21 chars
    'user' => [
        'id' => 'L066...',
        'name' => 'João Silva',
        'email' => 'joao@example.com',
        'role' => 'Administrador',
        'permissions' => ['Agenda.ver', 'Agenda.inserir'],
        'units' => [...],
    ],
    'expires_at' => 1234567890,
    'cookie_config' => [
        'name' => 'webdental_session_id',
        'domain' => '.webdental.local',
        'secure' => true,
        'httpOnly' => true,
        'sameSite' => 'None',
        'path' => '/',
    ],
]
```

**Exceções:**
- `SessionCreationFailedException` se não conseguir criar sessão

**Fluxo interno:**
```
1. Gera NanoID para session_id
2. Busca usuário no banco local pelo email (do ID token)
3. Carrega permissões e unidades do usuário
4. Salva sessão no Valkey com TTL
5. Salva refresh token separadamente
6. Adiciona session_id ao SET do usuário (para logout global)
7. Retorna dados da sessão + config do cookie
```

---

### getSession

Busca uma sessão pelo session_id.

```php
$session = $sessionService->getSession($sessionId);
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$sessionId` | string | NanoID da sessão |

**Retorno:**
```php
// Se encontrada:
[
    'sub' => 'cognito-uuid',
    'user_id' => 'L066...',
    'email' => 'joao@example.com',
    'name' => 'João Silva',
    'role' => 'Administrador',
    'permissions' => [...],
    'units' => [...],
    'id_token' => 'eyJ...',
    'access_token' => 'eyJ...',
    'created_at' => 1234567890,
    'expires_at' => 1234571490,
]

// Se não encontrada:
null
```

---

### refreshSession

Renova uma sessão usando o refresh token.

```php
$newSessionData = $sessionService->refreshSession($sessionId);
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$sessionId` | string | NanoID da sessão atual |

**Retorno:** Mesmo formato de `createSession()`

**Fluxo interno:**
```
1. Busca refresh token no Valkey
2. Chama Cognito para renovar tokens
3. Cria nova sessão com novos tokens
4. Remove sessão antiga
5. Retorna nova sessão com novo session_id
```

**Exceções:**
- `RefreshTokenNotFoundException` se refresh token não existe
- `RefreshTokenExpiredException` se refresh token expirou
- `CognitoException` se Cognito rejeitar o refresh

---

### invalidateSession

Remove uma sessão (logout simples).

```php
$sessionService->invalidateSession($sessionId);
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$sessionId` | string | NanoID da sessão |

**Retorno:** `void`

**Fluxo interno:**
```
1. Busca sessão para obter sub do usuário
2. Remove sessão do Valkey
3. Remove refresh token do Valkey
4. Remove session_id do SET do usuário
```

---

### invalidateAllSessions

Remove todas as sessões de um usuário (logout global).

```php
$sessionService->invalidateAllSessions($cognitoSub, $accessToken);
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$cognitoSub` | string | Sub (UUID) do usuário |
| `$accessToken` | string | Access token atual (opcional) |

**Retorno:** `void`

**Fluxo interno:**
```
1. Chama GlobalSignOut no Cognito (invalida refresh tokens)
2. Busca todas as session_ids do SET do usuário
3. Remove cada sessão do Valkey
4. Remove cada refresh token do Valkey
5. Remove o SET do usuário
```

---

### hasValidRefreshToken

Verifica se existe refresh token válido para uma sessão.

```php
$canRefresh = $sessionService->hasValidRefreshToken($sessionId);
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$sessionId` | string | NanoID da sessão |

**Retorno:** `bool`

---

## Estrutura de Dados no Valkey

### Sessão

**Chave:** `webdental:session:{nano_id}`

```json
{
    "sub": "abc-123-def",
    "user_id": "L066XXXXXXXXXXX",
    "email": "joao@example.com",
    "name": "João Silva",
    "role": "Administrador",
    "permissions": ["Agenda.ver", "Agenda.inserir", "..."],
    "units": [
        {"id": "U001", "name": "Unidade Centro"},
        {"id": "U002", "name": "Unidade Sul"}
    ],
    "id_token": "eyJhbGciOiJSUzI1NiIs...",
    "access_token": "eyJhbGciOiJSUzI1NiIs...",
    "created_at": 1234567890,
    "expires_at": 1234571490
}
```

**TTL:** Baseado no `expires_at` (geralmente 1 hora)

### Refresh Token

**Chave:** `webdental:refresh:{nano_id}`

```json
{
    "refresh_token": "eyJjdHkiOiJKV1QiLCJlbmMi...",
    "cognito_sub": "abc-123-def",
    "user_id": "L066XXXXXXXXXXX",
    "email": "joao@example.com",
    "expires_at": 1234567890,
    "created_at": 1234567890
}
```

**TTL:** 30 dias

### Mapeamento Usuário → Sessões

**Chave:** `webdental:user_sessions:{sub}`

**Tipo:** SET

```
{ "V1StGXR8_Z5jdHi6B-myT", "xYz789AbC_defGhi12345" }
```

**TTL:** Mesmo do refresh token mais longo

---

## Dependências

```
UserSessionService
├── SessionRepositoryInterface      → ValkeySessionRepository
├── RefreshTokenRepositoryInterface → ValkeyRefreshTokenRepository  
├── CognitoAuthServiceInterface     → CognitoAuthService
├── CognitoAdminServiceInterface    → CognitoAdminService
└── NanoIdGenerator
```

## Arquivos

| Arquivo | Localização |
|---------|-------------|
| Implementação | `app/SSO/Services/UserSessionService.php` |
| Interface | `app/SSO/Services/Contracts/UserSessionServiceInterface.php` |
| Testes | `tests/Unit/SSO/Services/UserSessionServiceTest.php` |
