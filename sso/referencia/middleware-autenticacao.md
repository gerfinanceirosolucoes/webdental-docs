# Middleware de Autenticação

## O que é

O `AuthenticateWithValkey` é o middleware que protege as rotas da API. Ele extrai o session_id do cookie, valida a sessão no Valkey e injeta os dados do usuário no request.

## Como funciona

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              REQUEST                                    │
│                                                                         │
│   Cookie: webdental_session_id=V1StGXR8_Z5jdHi6B-myT                   │
│                                                                         │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     AuthenticateWithValkey                              │
│                                                                         │
│   1. Extrai session_id do cookie                                        │
│   2. Busca sessão no Valkey                                             │
│   3. Verifica se não expirou                                            │
│   4. Injeta usuário no request                                          │
│                                                                         │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                    ┌────────────────┴────────────────┐
                    │                                 │
                    ▼                                 ▼
           ┌───────────────┐                 ┌───────────────┐
           │   SUCESSO     │                 │     ERRO      │
           │               │                 │               │
           │  Continua     │                 │  401 JSON     │
           │  para o       │                 │  com detalhes │
           │  Controller   │                 │               │
           └───────────────┘                 └───────────────┘
```

## Configuração

### Registrar no Kernel

```php
// app/Http/Kernel.php

protected $routeMiddleware = [
    // ...
    'sso.auth' => \App\SSO\Http\Middleware\AuthenticateWithValkey::class,
];
```

### Aplicar nas Rotas

```php
// routes/api.php

// Rota individual
Route::get('/user', 'UserController@show')
    ->middleware('sso.auth');

// Grupo de rotas
Route::middleware(['sso.auth'])->group(function () {
    Route::get('/agenda', 'AgendaController@index');
    Route::post('/paciente', 'PacienteController@store');
});
```

## Uso no Controller

Após o middleware, os dados do usuário estão disponíveis no request:

```php
class AgendaController extends Controller
{
    public function index(Request $request)
    {
        // Dados do usuário autenticado
        $user = $request->attributes->get('sso_user');
        
        $userId = $user['user_id'];
        $email = $user['email'];
        $role = $user['role'];
        $permissions = $user['permissions'];
        $units = $user['units'];
        
        // Verificar permissão
        if (!in_array('Agenda.ver', $permissions)) {
            abort(403, 'Sem permissão');
        }
        
        // ...
    }
}
```

### Estrutura do `sso_user`

```php
[
    'session_id' => 'V1StGXR8_Z5jdHi6B-myT',
    'sub' => 'abc-123-def',
    'user_id' => 'L066XXXXXXXXXXX',
    'email' => 'joao@example.com',
    'name' => 'João Silva',
    'role' => 'Administrador',
    'permissions' => [
        'Agenda.ver',
        'Agenda.inserir',
        'Agenda.editar',
        'Paciente.ver',
        // ...
    ],
    'units' => [
        ['id' => 'U001', 'name' => 'Unidade Centro'],
        ['id' => 'U002', 'name' => 'Unidade Sul'],
    ],
    'id_token' => 'eyJ...',
    'access_token' => 'eyJ...',
    'expires_at' => 1234571490,
]
```

## Respostas de Erro

### Sem Cookie

```json
{
    "error": "UNAUTHORIZED",
    "message": "Session ID não fornecido",
    "sso_expired": false,
    "can_refresh": false
}
```

**HTTP Status:** 401

### Sessão Não Encontrada

```json
{
    "error": "SESSION_NOT_FOUND",
    "message": "Sessão não encontrada",
    "sso_expired": true,
    "can_refresh": false
}
```

**HTTP Status:** 401

### Sessão Expirada (com refresh disponível)

```json
{
    "error": "SESSION_EXPIRED",
    "message": "Sessão expirada",
    "sso_expired": true,
    "can_refresh": true
}
```

**HTTP Status:** 401

**Ação do Frontend:** Chamar `POST /api/v1/auth/refresh` e retry

### Sessão Expirada (sem refresh)

```json
{
    "error": "SESSION_EXPIRED",
    "message": "Sessão expirada",
    "sso_expired": true,
    "can_refresh": false
}
```

**HTTP Status:** 401

**Ação do Frontend:** Redirecionar para login

## Rotas Excluídas

Algumas rotas não devem passar pelo middleware:

```php
// routes/api.php

// Rotas públicas (sem middleware)
Route::post('/v1/token/exchange', 'TokenExchangeController@exchange');
Route::post('/v1/auth/refresh', 'RefreshTokenController@refresh');

// Rotas protegidas (com middleware)
Route::middleware(['sso.auth'])->group(function () {
    Route::post('/v1/auth/logout', 'LogoutController@logout');
    Route::get('/v1/user', 'UserController@show');
    // ...
});
```

## Integração com Angular

O Angular deve enviar cookies em todas as requisições:

```typescript
// Interceptor
@Injectable()
export class CredentialsInterceptor implements HttpInterceptor {
    intercept(req: HttpRequest<any>, next: HttpHandler) {
        const cloned = req.clone({
            withCredentials: true
        });
        return next.handle(cloned);
    }
}
```

### Tratando 401 com Refresh

```typescript
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
    constructor(private authService: AuthService) {}

    intercept(req: HttpRequest<any>, next: HttpHandler) {
        return next.handle(req).pipe(
            catchError((error: HttpErrorResponse) => {
                if (error.status === 401 && error.error?.can_refresh) {
                    return this.authService.refresh().pipe(
                        switchMap(() => next.handle(req))
                    );
                }
                
                if (error.status === 401) {
                    this.authService.redirectToLogin();
                }
                
                return throwError(error);
            })
        );
    }
}
```

## Arquivo

| Arquivo | Localização |
|---------|-------------|
| Implementação | `app/SSO/Http/Middleware/AuthenticateWithValkey.php` |
