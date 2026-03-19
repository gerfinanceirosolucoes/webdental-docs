# Visão Geral da Arquitetura

## Contexto

O Webdental é composto por múltiplas aplicações que historicamente tinham sistemas de autenticação separados. O SSO unifica a autenticação usando AWS Cognito como provedor de identidade e Valkey como armazenamento de sessões.

## Diagrama de Alto Nível

```
                                    ┌─────────────────────┐
                                    │                     │
                                    │    AWS COGNITO      │
                                    │    (User Pool)      │
                                    │                     │
                                    │  • Autenticação     │
                                    │  • Tokens JWT       │
                                    │  • User Management  │
                                    │                     │
                                    └──────────┬──────────┘
                                               │
                                               │ OAuth 2.0 / OIDC
                                               │
┌──────────────────────────────────────────────┼──────────────────────────────────────────────┐
│                                              │                                              │
│                              INFRAESTRUTURA WEBDENTAL                                       │
│                                              │                                              │
│    ┌─────────────────────────────────────────┼─────────────────────────────────────────┐   │
│    │                                         ▼                                         │   │
│    │   ┌─────────────┐    Cookie      ┌─────────────┐      Valkey      ┌───────────┐  │   │
│    │   │             │◄──────────────►│             │◄────────────────►│           │  │   │
│    │   │   Angular   │                │   Laravel   │                  │   Valkey  │  │   │
│    │   │   (SPA)     │                │    API      │                  │  (Redis)  │  │   │
│    │   │             │                │             │                  │           │  │   │
│    │   └─────────────┘                └─────────────┘                  └───────────┘  │   │
│    │                                         ▲                                         │   │
│    │   ┌─────────────┐    Cookie             │                                         │   │
│    │   │             │◄──────────────────────┤                                         │   │
│    │   │  AngularJS  │                       │                                         │   │
│    │   │  (Legado)   │                       │                                         │   │
│    │   │             │                       │                                         │   │
│    │   └─────────────┘                       │                                         │   │
│    │                                         │                                         │   │
│    │   ┌─────────────┐    Cookie             │                                         │   │
│    │   │             │◄──────────────────────┘                                         │   │
│    │   │  Webdental  │                                                                 │   │
│    │   │ (PHP Legado)│                                                                 │   │
│    │   │             │                                                                 │   │
│    │   └─────────────┘                                                                 │   │
│    │                                                                                   │   │
│    │                         Domínio: .webdental.local                                 │   │
│    └───────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                            │
└────────────────────────────────────────────────────────────────────────────────────────────┘
```

## Componentes

### AWS Cognito (Conta AMEI)

**Responsabilidade:** Autenticação e gerenciamento de identidade

- User Pool compartilhado com outros sistemas (Webdental, AMEI)
- Emissão de tokens JWT (ID, Access, Refresh)
- Suporte a login com usuário/senha e Google (futuro)
- Managed Login UI customizada

**Localização:** Conta AWS AMEI

### Laravel API (Conta Webdental)

**Responsabilidade:** Orquestração da autenticação e autorização

- Troca tokens Cognito por sessão interna
- Validação de sessões
- Refresh de tokens
- Resolução de permissões do usuário no banco local

**Localização:** EC2 na conta AWS Webdental

### Valkey

**Responsabilidade:** Armazenamento de sessões

- Sessões com TTL automático
- Mapeamento usuário → sessões (para logout global)
- Alta performance para validação de cada request

**Estrutura de dados:**
```
webdental:session:{nano_id}     → Dados da sessão (JSON)
webdental:refresh:{nano_id}     → Refresh token (JSON)
webdental:user_sessions:{sub}   → SET de session_ids do usuário
```

### Aplicações Frontend

Todas as aplicações compartilham o mesmo cookie de sessão:

| Aplicação | Domínio | Tecnologia |
|-----------|---------|------------|
| Webdental (PHP) | `webdental.local` | PHP 7.0 |
| Angular | `ng.webdental.local` | Angular 6 |
| AngularJS | `angularjs.webdental.local` | AngularJS 1.x |
| API | `api.webdental.local` | Laravel 5.4 |

## Fluxo de Dados

### Login

```
1. Usuário acessa Webdental
2. Redireciona para Cognito Managed Login
3. Usuário autentica (usuário/senha)
4. Cognito redireciona de volta com authorization code
5. Backend troca code por tokens
6. Backend cria sessão no Valkey
7. Backend retorna cookie com session_id
8. Usuário acessa o sistema
```

### Request Autenticado

```
1. Browser envia cookie com session_id
2. Middleware extrai session_id
3. Middleware busca sessão no Valkey
4. Se válida, injeta usuário no request
5. Controller processa a requisição
6. Retorna resposta
```

### Logout

```
1. Usuário clica em "Sair"
2. Backend chama GlobalSignOut no Cognito
3. Backend remove sessão do Valkey
4. Backend remove cookie
5. Redireciona para tela de login
```

## Segurança

| Aspecto | Implementação |
|---------|---------------|
| **Armazenamento de sessão** | Cookie HTTPOnly, Secure, SameSite=None |
| **Proteção XSS** | Tokens nunca expostos ao JavaScript |
| **Proteção CSRF** | SameSite + validação de origem |
| **Expiração** | TTL automático no Valkey |
| **Logout global** | GlobalSignOut invalida todos os refresh tokens |
| **Credenciais AWS** | Rotação automática via Secrets Manager |

## Contas AWS

| Recurso | Conta |
|---------|-------|
| Cognito, Secrets Manager | AMEI |
| EC2, Valkey | Webdental |

O acesso cross-account é feito via credenciais IAM fixas que acessam o Secrets Manager para buscar credenciais rotacionadas.
