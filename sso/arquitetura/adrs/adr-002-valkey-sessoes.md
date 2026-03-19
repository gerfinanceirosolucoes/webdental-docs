# ADR-002: Valkey para Sessões

## Status

✅ Aceito

## Data

Fevereiro 2025

## Contexto

Precisamos armazenar sessões de usuários de forma que:

1. Seja acessível por múltiplas instâncias da aplicação (load balancer)
2. Tenha expiração automática (TTL)
3. Permita invalidação imediata (logout)
4. Suporte consultas rápidas (validação em cada request)
5. Permita listar todas as sessões de um usuário (logout global)

O Cognito fornece tokens JWT, mas precisamos de um estado de sessão interno para:
- Armazenar permissões do usuário (que vêm do banco local)
- Controlar sessões ativas
- Permitir invalidação sem depender do Cognito

## Decisão

Usar **Valkey** (fork do Redis compatível) como armazenamento de sessões.

### Estrutura de dados

```
# Sessão principal (expira junto com access_token)
webdental:session:{nano_id} = {
    "sub": "cognito-uuid",
    "user_id": "L066...",
    "email": "user@example.com",
    "name": "Nome",
    "role": "Administrador",
    "permissions": ["Agenda.ver", "Agenda.inserir"],
    "id_token": "eyJ...",
    "access_token": "eyJ...",
    "refresh_token": "eyJ...",
    "created_at": 1234567890,
    "expires_at": 1234571490
}
EXPIREAT: timestamp do access_token

# Refresh token (expira em 30 dias)
webdental:refresh:{nano_id} = {
    "refresh_token": "eyJ...",
    "cognito_sub": "cognito-uuid",
    "user_id": "L066...",
    "email": "user@example.com",
    "expires_at": 1234567890,
    "created_at": 1234567890
}
EXPIREAT: timestamp do refresh_token

# Mapeamento usuário → sessões (para logout global)
webdental:user_sessions:{sub} = SET { nano_id_1, nano_id_2, ... }
TTL: mesmo do refresh_token mais longo
```

### Por que separar sessão e refresh token?

| Dado | TTL | Motivo |
|------|-----|--------|
| Sessão | 20 minutos | Expira junto com access_token |
| Refresh Token | 8 horas | Permite renovar sem re-login |

Quando o access_token expira:
1. Sessão é removida automaticamente pelo Valkey
2. Refresh token ainda existe
3. Cliente chama endpoint de refresh
4. Nova sessão é criada com novo NanoID

## Consequências

### Positivas

- **Performance**: Validação de sessão em < 1ms
- **Escalabilidade**: Suporta múltiplas instâncias da API
- **TTL automático**: Limpeza automática de sessões expiradas
- **Logout global**: Fácil listar e invalidar todas as sessões de um usuário
- **Compatibilidade**: Redis-compatible, muitas ferramentas disponíveis

### Negativas

- **Dependência adicional**: Mais um serviço para gerenciar
- **Custo**: Instância Valkey em produção
- **Persistência**: Por padrão em memória (pode perder dados em restart)

## Alternativas Consideradas

### Banco de dados (MariaDB)

**Rejeitado porque:**
- Latência maior (disco vs memória)
- Não tem TTL nativo
- Carga adicional no banco principal

### JWT stateless

**Rejeitado porque:**
- Não permite invalidação antes da expiração
- Não suporta logout global
- Token grande em cada request

### PHP Sessions

**Rejeitado porque:**
- Não compartilha entre instâncias facilmente
- Não funciona com load balancer sem sticky sessions
- Difícil de integrar com Angular/AngularJS

## Configuração

```php
// config/sso.php
'valkey' => [
    'host' => env('VALKEY_HOST', '127.0.0.1'),
    'port' => env('VALKEY_PORT', 6379),
    'password' => env('VALKEY_PASSWORD', null),
    'database' => env('VALKEY_DATABASE', 0),
    'session' => [
        'prefix' => 'webdental:session:',
        'user_prefix' => 'webdental:user_sessions:',
    ],
    'refresh' => [
        'prefix' => 'webdental:refresh:',
        'ttl' => 2592000, // 30 dias
    ],
],
```
