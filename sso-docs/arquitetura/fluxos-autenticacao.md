# Fluxos de Autenticação

## Fluxo 1: Login Inicial

Quando o usuário acessa o sistema sem estar autenticado.

```
┌─────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────┐     ┌─────────┐
│ Browser │     │  Webdental  │     │    API      │     │ Cognito │     │ Valkey  │
└────┬────┘     └──────┬──────┘     └──────┬──────┘     └────┬────┘     └────┬────┘
     │                 │                   │                 │               │
     │  1. Acessa /    │                   │                 │               │
     │────────────────►│                   │                 │               │
     │                 │                   │                 │               │
     │                 │ 2. Sem sessão,    │                 │               │
     │                 │    redireciona    │                 │               │
     │◄────────────────│                   │                 │               │
     │  302 → Cognito  │                   │                 │               │
     │                 │                   │                 │               │
     │  3. Tela de login Cognito           │                 │               │
     │─────────────────────────────────────────────────────►│               │
     │                 │                   │                 │               │
     │  4. Usuário preenche credenciais    │                 │               │
     │─────────────────────────────────────────────────────►│               │
     │                 │                   │                 │               │
     │  5. Cognito valida e redireciona    │                 │               │
     │◄─────────────────────────────────────────────────────│               │
     │  302 → /sso/callback?code=xxx       │                 │               │
     │                 │                   │                 │               │
     │  6. Callback    │                   │                 │               │
     │────────────────►│                   │                 │               │
     │                 │                   │                 │               │
     │                 │ 7. POST /token/exchange             │               │
     │                 │──────────────────►│                 │               │
     │                 │                   │                 │               │
     │                 │                   │ 8. Troca code   │               │
     │                 │                   │    por tokens   │               │
     │                 │                   │────────────────►│               │
     │                 │                   │◄────────────────│               │
     │                 │                   │  id, access,    │               │
     │                 │                   │  refresh tokens │               │
     │                 │                   │                 │               │
     │                 │                   │ 9. Busca user   │               │
     │                 │                   │    no banco     │               │
     │                 │                   │    local        │               │
     │                 │                   │                 │               │
     │                 │                   │ 10. Cria sessão │               │
     │                 │                   │────────────────────────────────►│
     │                 │                   │◄────────────────────────────────│
     │                 │                   │                 │               │
     │                 │◄──────────────────│                 │               │
     │                 │ session_id +      │                 │               │
     │                 │ config do cookie  │                 │               │
     │                 │                   │                 │               │
     │◄────────────────│                   │                 │               │
     │ 11. Set-Cookie: │                   │                 │               │
     │     webdental_  │                   │                 │               │
     │     session_id  │                   │                 │               │
     │                 │                   │                 │               │
     │ 12. Redireciona │                   │                 │               │
     │     para home   │                   │                 │               │
     │────────────────►│                   │                 │               │
     │                 │                   │                 │               │
```

## Fluxo 2: Request Autenticado

Quando o usuário já está logado e faz uma requisição.

```
┌─────────┐     ┌─────────┐     ┌────────────┐     ┌─────────┐
│ Browser │     │   API   │     │ Middleware │     │ Valkey  │
└────┬────┘     └────┬────┘     └─────┬──────┘     └────┬────┘
     │               │                │                 │
     │ 1. Request    │                │                 │
     │    + Cookie   │                │                 │
     │──────────────►│                │                 │
     │               │                │                 │
     │               │ 2. Extrai      │                 │
     │               │    session_id  │                 │
     │               │───────────────►│                 │
     │               │                │                 │
     │               │                │ 3. Busca sessão │
     │               │                │────────────────►│
     │               │                │◄────────────────│
     │               │                │   dados user    │
     │               │                │                 │
     │               │                │ 4. Valida       │
     │               │                │    expiração    │
     │               │                │                 │
     │               │◄───────────────│                 │
     │               │ 5. Injeta user │                 │
     │               │    no request  │                 │
     │               │                │                 │
     │               │ 6. Controller  │                 │
     │               │    processa    │                 │
     │               │                │                 │
     │◄──────────────│                │                 │
     │ 7. Response   │                │                 │
     │               │                │                 │
```

## Fluxo 3: Refresh de Sessão

Quando o access_token expira mas o refresh_token ainda é válido.

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Angular │     │   API   │     │ Cognito │     │ Valkey  │
└────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘
     │               │               │               │
     │ 1. Request    │               │               │
     │──────────────►│               │               │
     │               │               │               │
     │◄──────────────│               │               │
     │ 401 + can_    │               │               │
     │ refresh: true │               │               │
     │               │               │               │
     │ 2. POST       │               │               │
     │ /auth/refresh │               │               │
     │──────────────►│               │               │
     │               │               │               │
     │               │ 3. Busca      │               │
     │               │    refresh    │               │
     │               │    token      │               │
     │               │──────────────────────────────►│
     │               │◄──────────────────────────────│
     │               │               │               │
     │               │ 4. Renova     │               │
     │               │    tokens     │               │
     │               │──────────────►│               │
     │               │◄──────────────│               │
     │               │  novos tokens │               │
     │               │               │               │
     │               │ 5. Cria nova  │               │
     │               │    sessão     │               │
     │               │──────────────────────────────►│
     │               │               │               │
     │               │ 6. Remove     │               │
     │               │    sessão     │               │
     │               │    antiga     │               │
     │               │──────────────────────────────►│
     │               │               │               │
     │◄──────────────│               │               │
     │ Set-Cookie:   │               │               │
     │ novo session  │               │               │
     │               │               │               │
     │ 7. Retry      │               │               │
     │    request    │               │               │
     │    original   │               │               │
     │──────────────►│               │               │
     │               │               │               │
```

## Fluxo 4: Logout

Quando o usuário clica em "Sair".

```
┌─────────┐     ┌─────────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Browser │     │  Webdental  │     │   API   │     │ Cognito │     │ Valkey  │
└────┬────┘     └──────┬──────┘     └────┬────┘     └────┬────┘     └────┬────┘
     │                 │                 │               │               │
     │ 1. Clica Sair   │                 │               │               │
     │────────────────►│                 │               │               │
     │                 │                 │               │               │
     │                 │ 2. POST /logout │               │               │
     │                 │────────────────►│               │               │
     │                 │                 │               │               │
     │                 │                 │ 3. GlobalSign │               │
     │                 │                 │    Out        │               │
     │                 │                 │──────────────►│               │
     │                 │                 │◄──────────────│               │
     │                 │                 │ (invalida     │               │
     │                 │                 │  refresh      │               │
     │                 │                 │  tokens)      │               │
     │                 │                 │               │               │
     │                 │                 │ 4. Remove     │               │
     │                 │                 │    sessão     │               │
     │                 │                 │──────────────────────────────►│
     │                 │                 │               │               │
     │                 │                 │ 5. Remove do  │               │
     │                 │                 │    SET user   │               │
     │                 │                 │──────────────────────────────►│
     │                 │                 │               │               │
     │                 │◄────────────────│               │               │
     │                 │  clear cookie   │               │               │
     │                 │                 │               │               │
     │◄────────────────│                 │               │               │
     │ Set-Cookie:     │                 │               │               │
     │ Max-Age=0       │                 │               │               │
     │                 │                 │               │               │
     │ 6. Redireciona  │                 │               │               │
     │    para login   │                 │               │               │
     │◄────────────────│                 │               │               │
     │                 │                 │               │               │
```

## Fluxo 5: Usuário não encontrado no Webdental

Quando o usuário existe no Cognito mas não no banco local do Webdental.

```
┌─────────┐     ┌─────────────┐     ┌─────────┐     ┌──────────┐
│ Browser │     │  Webdental  │     │   API   │     │  Banco   │
└────┬────┘     └──────┬──────┘     └────┬────┘     └────┬─────┘
     │                 │                 │               │
     │ (após callback  │                 │               │
     │  do Cognito)    │                 │               │
     │                 │                 │               │
     │                 │ 1. token/       │               │
     │                 │    exchange     │               │
     │                 │────────────────►│               │
     │                 │                 │               │
     │                 │                 │ 2. Busca user │
     │                 │                 │    por email  │
     │                 │                 │──────────────►│
     │                 │                 │◄──────────────│
     │                 │                 │  NOT FOUND    │
     │                 │                 │               │
     │                 │◄────────────────│               │
     │                 │ 404 USER_NOT_   │               │
     │                 │     FOUND       │               │
     │                 │                 │               │
     │◄────────────────│                 │               │
     │ Redireciona     │                 │               │
     │ /sso/user-not-  │                 │               │
     │ found.php       │                 │               │
     │                 │                 │               │
     │ 3. Exibe        │                 │               │
     │    mensagem     │                 │               │
     │    de erro      │                 │               │
     │                 │                 │               │
```

## Códigos de Erro

| Código | HTTP | Descrição | Ação do Frontend |
|--------|------|-----------|------------------|
| `SESSION_EXPIRED` | 401 | Sessão expirou | Tentar refresh ou redirecionar para login |
| `UNAUTHORIZED` | 401 | Sem session_id | Redirecionar para login |
| `USER_NOT_FOUND` | 404 | Usuário não existe no Webdental | Exibir página de erro |
| `USER_INACTIVE` | 403 | Usuário inativo | Exibir página de erro |
| `NO_UNITS` | 403 | Usuário sem unidades | Exibir página de erro |
