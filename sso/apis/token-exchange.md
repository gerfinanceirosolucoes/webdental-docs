# POST /api/v1/token/exchange

## Descrição

Troca o authorization code do Cognito por uma sessão do Webdental. Este endpoint é chamado pelo callback do SSO após o usuário autenticar no Cognito.

## Endpoint

```
POST /api/v1/token/exchange
```

## Autenticação

Não requer autenticação (é o endpoint que cria a sessão).

## Request

### Headers

```
Content-Type: application/json
```

### Body

```json
{
    "code": "abc123-authorization-code-from-cognito"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `code` | string | Sim | Authorization code retornado pelo Cognito |

## Response

### Sucesso (200)

```json
{
    "success": true,
    "data": {
        "session_id": "V1StGXR8_Z5jdHi6B-myT",
        "user": {
            "id": "L066XXXXXXXXXXX",
            "name": "João Silva",
            "email": "joao@example.com",
            "role": "Administrador",
            "permissions": [
                "Agenda.ver",
                "Agenda.inserir",
                "Paciente.ver"
            ],
            "units": [
                {
                    "id": "U001",
                    "name": "Unidade Centro"
                }
            ]
        },
        "expires_at": 1234571490
    },
    "cookie": {
        "name": "webdental_session_id",
        "value": "V1StGXR8_Z5jdHi6B-myT",
        "domain": ".webdental.local",
        "secure": true,
        "httpOnly": true,
        "sameSite": "None",
        "path": "/"
    }
}
```

O response inclui `Set-Cookie` header para setar o cookie automaticamente.

### Erro: Code inválido (400)

```json
{
    "success": false,
    "error": "INVALID_CODE",
    "message": "Authorization code inválido ou expirado"
}
```

### Erro: Usuário não encontrado (404)

```json
{
    "success": false,
    "error": "USER_NOT_FOUND",
    "message": "Usuário não encontrado no Webdental",
    "cognito_email": "joao@example.com"
}
```

**Causa:** Usuário autenticou no Cognito mas não existe na tabela `tbl_prestador`.

### Erro: Usuário inativo (403)

```json
{
    "success": false,
    "error": "USER_INACTIVE",
    "message": "Usuário inativo no sistema"
}
```

### Erro: Usuário sem unidades (403)

```json
{
    "success": false,
    "error": "NO_UNITS",
    "message": "Usuário não possui unidades associadas"
}
```

## Fluxo

```
┌──────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐
│ Frontend │      │   API   │      │ Cognito │      │  Banco  │      │ Valkey  │
└────┬─────┘      └────┬────┘      └────┬────┘      └────┬────┘      └────┬────┘
     │                 │                │                │                │
     │ POST /exchange  │                │                │                │
     │ { code: "xxx" } │                │                │                │
     │────────────────►│                │                │                │
     │                 │                │                │                │
     │                 │ POST /token    │                │                │
     │                 │ (troca code)   │                │                │
     │                 │───────────────►│                │                │
     │                 │◄───────────────│                │                │
     │                 │ id, access,    │                │                │
     │                 │ refresh tokens │                │                │
     │                 │                │                │                │
     │                 │ Busca usuário  │                │                │
     │                 │ pelo email     │                │                │
     │                 │───────────────────────────────►│                │
     │                 │◄───────────────────────────────│                │
     │                 │                │                │                │
     │                 │ Cria sessão    │                │                │
     │                 │────────────────────────────────────────────────►│
     │                 │◄────────────────────────────────────────────────│
     │                 │                │                │                │
     │◄────────────────│                │                │                │
     │ { session_id,   │                │                │                │
     │   user, cookie }│                │                │                │
     │ + Set-Cookie    │                │                │                │
     │                 │                │                │                │
```

## Exemplo de Uso

### JavaScript (no callback.php)

```javascript
async function exchangeCode(code) {
    const response = await fetch('/api/v1/token/exchange', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        credentials: 'include', // Importante para receber o cookie
        body: JSON.stringify({ code }),
    });

    if (!response.ok) {
        const error = await response.json();
        
        if (error.error === 'USER_NOT_FOUND') {
            window.location.href = '/sso/user-not-found.php';
            return;
        }
        
        throw new Error(error.message);
    }

    const data = await response.json();
    
    // Redireciona para home
    window.location.href = '/';
}

// Extrair code da URL
const urlParams = new URLSearchParams(window.location.search);
const code = urlParams.get('code');

if (code) {
    exchangeCode(code);
}
```

### PHP (no callback.php)

```php
<?php
$code = $_GET['code'] ?? null;

if (!$code) {
    header('Location: /sso/auth.php');
    exit;
}

$response = file_get_contents('https://api.webdental.local/api/v1/token/exchange', false, stream_context_create([
    'http' => [
        'method' => 'POST',
        'header' => 'Content-Type: application/json',
        'content' => json_encode(['code' => $code]),
    ],
]));

$data = json_decode($response, true);

if (!$data['success']) {
    if ($data['error'] === 'USER_NOT_FOUND') {
        header('Location: /sso/user-not-found.php');
        exit;
    }
    
    header('Location: /sso/error.php?msg=' . urlencode($data['message']));
    exit;
}

// Cookie já foi setado pelo header Set-Cookie
// Redireciona para home
header('Location: /');
```
