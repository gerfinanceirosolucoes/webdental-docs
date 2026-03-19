# POST /api/v1/auth/refresh

## Descrição

Renova a sessão usando o refresh token. Deve ser chamado quando o access token expira mas o refresh token ainda é válido.

## Endpoint

```
POST /api/v1/auth/refresh
```

## Autenticação

Requer o cookie `webdental_session_id` com session_id válido (mesmo que expirado).

## Request

### Headers

```
Content-Type: application/json
Cookie: webdental_session_id=V1StGXR8_Z5jdHi6B-myT
```

### Body

Nenhum body necessário. O session_id é extraído do cookie.

```json
{}
```

## Response

### Sucesso (200)

```json
{
    "success": true,
    "data": {
        "session_id": "xYz789AbC_defGhi12345",
        "user": {
            "id": "L066XXXXXXXXXXX",
            "name": "João Silva",
            "email": "joao@example.com",
            "role": "Administrador",
            "permissions": [
                "Agenda.ver",
                "Agenda.inserir"
            ],
            "units": [
                {
                    "id": "U001",
                    "name": "Unidade Centro"
                }
            ]
        },
        "expires_at": 1234578890
    }
}
```

**Importante:** O `session_id` é **novo**. O cookie é atualizado automaticamente via `Set-Cookie` header.

### Erro: Sem refresh token (401)

```json
{
    "success": false,
    "error": "REFRESH_TOKEN_NOT_FOUND",
    "message": "Refresh token não encontrado",
    "sso_expired": true,
    "can_refresh": false
}
```

**Ação:** Redirecionar para login.

### Erro: Refresh token expirado (401)

```json
{
    "success": false,
    "error": "REFRESH_TOKEN_EXPIRED",
    "message": "Refresh token expirado",
    "sso_expired": true,
    "can_refresh": false
}
```

**Ação:** Redirecionar para login.

### Erro: Cognito rejeitou refresh (401)

```json
{
    "success": false,
    "error": "COGNITO_REFRESH_FAILED",
    "message": "Não foi possível renovar a sessão",
    "sso_expired": true,
    "can_refresh": false
}
```

**Causa possível:** GlobalSignOut foi chamado, invalidando todos os refresh tokens.

**Ação:** Redirecionar para login.

## Fluxo

```
┌──────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐
│  Angular │      │   API   │      │ Cognito │      │ Valkey  │
└────┬─────┘      └────┬────┘      └────┬────┘      └────┬────┘
     │                 │                │                │
     │ POST /refresh   │                │                │
     │ Cookie: xxx     │                │                │
     │────────────────►│                │                │
     │                 │                │                │
     │                 │ Busca refresh  │                │
     │                 │ token          │                │
     │                 │───────────────────────────────►│
     │                 │◄───────────────────────────────│
     │                 │                │                │
     │                 │ POST /token    │                │
     │                 │ grant_type=    │                │
     │                 │ refresh_token  │                │
     │                 │───────────────►│                │
     │                 │◄───────────────│                │
     │                 │ novos tokens   │                │
     │                 │                │                │
     │                 │ Cria nova      │                │
     │                 │ sessão         │                │
     │                 │───────────────────────────────►│
     │                 │                │                │
     │                 │ Remove sessão  │                │
     │                 │ antiga         │                │
     │                 │───────────────────────────────►│
     │                 │                │                │
     │◄────────────────│                │                │
     │ { session_id }  │                │                │
     │ + Set-Cookie    │                │                │
     │                 │                │                │
```

## Exemplo de Uso

### Angular Service

```typescript
@Injectable({
    providedIn: 'root'
})
export class AuthService {
    constructor(private http: HttpClient) {}

    refresh(): Observable<RefreshResponse> {
        return this.http.post<RefreshResponse>(
            '/api/v1/auth/refresh',
            {},
            { withCredentials: true }
        );
    }
}
```

### Angular Interceptor

```typescript
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
    private isRefreshing = false;
    private refreshSubject = new BehaviorSubject<boolean>(false);

    constructor(private authService: AuthService, private router: Router) {}

    intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
        return next.handle(req).pipe(
            catchError((error: HttpErrorResponse) => {
                if (error.status === 401) {
                    return this.handle401Error(req, next, error);
                }
                return throwError(error);
            })
        );
    }

    private handle401Error(
        req: HttpRequest<any>,
        next: HttpHandler,
        error: HttpErrorResponse
    ): Observable<HttpEvent<any>> {
        
        // Verifica se pode fazer refresh
        if (!error.error?.can_refresh) {
            this.router.navigate(['/login']);
            return throwError(error);
        }

        // Se já está fazendo refresh, espera
        if (this.isRefreshing) {
            return this.refreshSubject.pipe(
                filter(result => result === true),
                take(1),
                switchMap(() => next.handle(req))
            );
        }

        this.isRefreshing = true;
        this.refreshSubject.next(false);

        return this.authService.refresh().pipe(
            switchMap(() => {
                this.isRefreshing = false;
                this.refreshSubject.next(true);
                return next.handle(req);
            }),
            catchError((refreshError) => {
                this.isRefreshing = false;
                this.router.navigate(['/login']);
                return throwError(refreshError);
            })
        );
    }
}
```

## Comportamento Esperado

| Situação | can_refresh | Ação |
|----------|-------------|------|
| Access token expirou, refresh token válido | `true` | Chamar `/auth/refresh` |
| Access token expirou, refresh token expirou | `false` | Redirecionar para login |
| GlobalSignOut foi chamado | `false` | Redirecionar para login |
| Sessão não existe no Valkey | `false` | Redirecionar para login |

## Notas

- O refresh gera um **novo session_id**. O cookie antigo é substituído.
- O refresh token no Cognito tem TTL de 30 dias por padrão.
- Se o usuário fez logout em outro dispositivo (GlobalSignOut), o refresh vai falhar.
- O endpoint não requer body, todo o contexto vem do cookie.
