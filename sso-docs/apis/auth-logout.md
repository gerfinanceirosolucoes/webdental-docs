# POST /api/v1/auth/logout

## Descrição

Encerra a sessão do usuário. Remove a sessão do Valkey, invalida os tokens no Cognito e limpa o cookie.

## Endpoint

```
POST /api/v1/auth/logout
```

## Autenticação

Requer o cookie `webdental_session_id` com session_id válido.

## Request

### Headers

```
Content-Type: application/json
Cookie: webdental_session_id=V1StGXR8_Z5jdHi6B-myT
```

### Body

```json
{
    "global": false
}
```

| Campo | Tipo | Obrigatório | Default | Descrição |
|-------|------|-------------|---------|-----------|
| `global` | boolean | Não | `true` | Se `true`, faz logout de todos os dispositivos |

## Response

### Sucesso (200)

```json
{
    "success": true,
    "message": "Logout realizado com sucesso",
    "redirect_url": "https://webdental.local/sso/auth.php"
}
```

O response inclui `Set-Cookie` header para limpar o cookie:

```
Set-Cookie: webdental_session_id=; Max-Age=0; Domain=.webdental.local; Path=/; Secure; HttpOnly; SameSite=None
```

### Erro: Sessão não encontrada (401)

```json
{
    "success": false,
    "error": "SESSION_NOT_FOUND",
    "message": "Sessão não encontrada"
}
```

**Ação:** Redirecionar para login (já está deslogado).

## Fluxo

### Logout Simples (global: false)

Remove apenas a sessão atual. Outras sessões do usuário continuam válidas.

```
┌──────────┐      ┌─────────┐      ┌─────────┐
│  Browser │      │   API   │      │ Valkey  │
└────┬─────┘      └────┬────┘      └────┬────┘
     │                 │                │
     │ POST /logout    │                │
     │ global: false   │                │
     │────────────────►│                │
     │                 │                │
     │                 │ Remove sessão  │
     │                 │───────────────►│
     │                 │                │
     │                 │ Remove refresh │
     │                 │───────────────►│
     │                 │                │
     │◄────────────────│                │
     │ Set-Cookie: ""  │                │
     │ Max-Age=0       │                │
     │                 │                │
```

### Logout Global (global: true)

Remove todas as sessões e invalida todos os refresh tokens no Cognito.

```
┌──────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐
│  Browser │      │   API   │      │ Cognito │      │ Valkey  │
└────┬─────┘      └────┬────┘      └────┬────┘      └────┬────┘
     │                 │                │                │
     │ POST /logout    │                │                │
     │ global: true    │                │                │
     │────────────────►│                │                │
     │                 │                │                │
     │                 │ GlobalSignOut  │                │
     │                 │───────────────►│                │
     │                 │◄───────────────│                │
     │                 │ (invalida      │                │
     │                 │  todos refresh │                │
     │                 │  tokens)       │                │
     │                 │                │                │
     │                 │ Busca todas    │                │
     │                 │ sessões user   │                │
     │                 │───────────────────────────────►│
     │                 │◄───────────────────────────────│
     │                 │                │                │
     │                 │ Remove cada    │                │
     │                 │ sessão         │                │
     │                 │───────────────────────────────►│
     │                 │                │                │
     │◄────────────────│                │                │
     │ Set-Cookie: ""  │                │                │
     │ Max-Age=0       │                │                │
     │                 │                │                │
```

## Exemplo de Uso

### Angular Service

```typescript
@Injectable({
    providedIn: 'root'
})
export class AuthService {
    constructor(
        private http: HttpClient,
        private router: Router
    ) {}

    logout(global: boolean = true): Observable<void> {
        return this.http.post<LogoutResponse>(
            '/api/v1/auth/logout',
            { global },
            { withCredentials: true }
        ).pipe(
            tap(response => {
                // Redireciona para login
                window.location.href = response.redirect_url;
            }),
            map(() => void 0)
        );
    }
}
```

### PHP (Webdental Legado)

```php
<?php

function logout($global = true) {
    $ch = curl_init();
    
    curl_setopt_array($ch, [
        CURLOPT_URL => 'https://api.webdental.local/api/v1/auth/logout',
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode(['global' => $global]),
        CURLOPT_HTTPHEADER => [
            'Content-Type: application/json',
            'Cookie: webdental_session_id=' . $_COOKIE['webdental_session_id'],
        ],
        CURLOPT_RETURNTRANSFER => true,
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $data = json_decode($response, true);
    
    // Limpa cookie local também
    setcookie('webdental_session_id', '', [
        'expires' => time() - 3600,
        'path' => '/',
        'domain' => '.webdental.local',
        'secure' => true,
        'httponly' => true,
        'samesite' => 'None',
    ]);
    
    // Redireciona
    header('Location: ' . $data['redirect_url']);
    exit;
}
```

### JavaScript

```javascript
async function logout(global = true) {
    try {
        const response = await fetch('/api/v1/auth/logout', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            credentials: 'include',
            body: JSON.stringify({ global }),
        });

        const data = await response.json();

        if (data.success) {
            window.location.href = data.redirect_url;
        }
    } catch (error) {
        // Em caso de erro, redireciona para login mesmo assim
        window.location.href = '/sso/auth.php';
    }
}
```

## Comportamento

| Parâmetro `global` | Comportamento |
|--------------------|---------------|
| `true` (default) | Remove TODAS as sessões do usuário em todos os dispositivos |
| `false` | Remove apenas a sessão atual |

## Quando usar cada opção

| Cenário | `global` |
|---------|----------|
| Usuário clica em "Sair" | `true` |
| Sessão expira no frontend | `false` |
| Admin desativa usuário | `true` (via adminUserGlobalSignOut) |
| Usuário troca senha | `true` |
| Usuário esqueceu dispositivo em local público | `true` |

## Notas

- O logout **sempre** limpa o cookie, mesmo se falhar em outras operações.
- Se o access token já expirou, usa `adminUserGlobalSignOut` que não precisa do token.
- O `redirect_url` retornado é configurável em `config/sso.php`.
- Após o logout, o usuário precisa autenticar novamente no Cognito.
