# Diagrama de Componentes

## Visão Geral

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                   FRONTEND                                       │
├─────────────────────┬─────────────────────┬─────────────────────────────────────┤
│                     │                     │                                     │
│   ┌─────────────┐   │   ┌─────────────┐   │   ┌─────────────────────────────┐   │
│   │             │   │   │             │   │   │                             │   │
│   │  Webdental  │   │   │   Angular   │   │   │         AngularJS           │   │
│   │    (PHP)    │   │   │             │   │   │                             │   │
│   │             │   │   │             │   │   │                             │   │
│   └──────┬──────┘   │   └──────┬──────┘   │   └──────────────┬──────────────┘   │
│          │          │          │          │                  │                  │
└──────────┼──────────┴──────────┼──────────┴──────────────────┼──────────────────┘
           │                     │                             │
           │     Cookie: webdental_session_id                  │
           │                     │                             │
           └─────────────────────┼─────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              LARAVEL API                                         │
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                            MIDDLEWARE                                     │  │
│   │                                                                          │  │
│   │   ┌─────────────────────┐    ┌─────────────────────────────────────┐    │  │
│   │   │                     │    │                                     │    │  │
│   │   │  AuthenticateWith   │───►│         UserSessionService          │    │  │
│   │   │      Valkey         │    │                                     │    │  │
│   │   │                     │    │  • getSession()                     │    │  │
│   │   └─────────────────────┘    │  • createSession()                  │    │  │
│   │                              │  • invalidateSession()              │    │  │
│   │                              │  • refreshSession()                 │    │  │
│   │                              │                                     │    │  │
│   │                              └──────────────┬──────────────────────┘    │  │
│   │                                             │                           │  │
│   └─────────────────────────────────────────────┼───────────────────────────┘  │
│                                                 │                              │
│   ┌─────────────────────────────────────────────┼───────────────────────────┐  │
│   │                          SERVICES           │                           │  │
│   │                                             │                           │  │
│   │   ┌─────────────────────┐    ┌──────────────┴──────────────┐           │  │
│   │   │                     │    │                             │           │  │
│   │   │  CognitoAuthService │    │    CognitoAdminService      │           │  │
│   │   │                     │    │                             │           │  │
│   │   │  • exchangeCode()   │    │  • adminGetUser()           │           │  │
│   │   │  • refreshTokens()  │    │  • globalSignOut()          │           │  │
│   │   │  • validateToken()  │    │  • adminUserGlobalSignOut() │           │  │
│   │   │                     │    │  • adminCreateUser()        │           │  │
│   │   └──────────┬──────────┘    │  • adminDisableUser()       │           │  │
│   │              │               │                             │           │  │
│   │              │               └──────────────┬──────────────┘           │  │
│   │              │                              │                          │  │
│   │              │               ┌──────────────┴──────────────┐           │  │
│   │              │               │                             │           │  │
│   │              │               │   AwsCredentialsService     │           │  │
│   │              │               │                             │           │  │
│   │              │               │  • getCognitoAdminCreds()   │           │  │
│   │              │               │                             │           │  │
│   │              │               └──────────────┬──────────────┘           │  │
│   │              │                              │                          │  │
│   └──────────────┼──────────────────────────────┼──────────────────────────┘  │
│                  │                              │                             │
│   ┌──────────────┼──────────────────────────────┼──────────────────────────┐  │
│   │              │     REPOSITORIES             │                          │  │
│   │              │                              │                          │  │
│   │   ┌──────────┴──────────┐    ┌──────────────┴──────────────┐          │  │
│   │   │                     │    │                             │          │  │
│   │   │  SessionRepository  │    │  RefreshTokenRepository     │          │  │
│   │   │                     │    │                             │          │  │
│   │   └──────────┬──────────┘    └──────────────┬──────────────┘          │  │
│   │              │                              │                          │  │
│   └──────────────┼──────────────────────────────┼──────────────────────────┘  │
│                  │                              │                             │
└──────────────────┼──────────────────────────────┼─────────────────────────────┘
                   │                              │
                   ▼                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                  VALKEY                                          │
│                                                                                  │
│   ┌─────────────────────────────┐    ┌─────────────────────────────────────┐   │
│   │                             │    │                                     │   │
│   │  webdental:session:{id}     │    │  webdental:refresh:{id}             │   │
│   │                             │    │                                     │   │
│   │  TTL: access_token exp      │    │  TTL: 30 dias                       │   │
│   │                             │    │                                     │   │
│   └─────────────────────────────┘    └─────────────────────────────────────┘   │
│                                                                                  │
│   ┌─────────────────────────────┐                                               │
│   │                             │                                               │
│   │  webdental:user_sessions:   │                                               │
│   │  {sub} (SET)                │                                               │
│   │                             │                                               │
│   └─────────────────────────────┘                                               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

                   │
                   │ (Chamadas externas)
                   ▼

┌─────────────────────────────────────────────────────────────────────────────────┐
│                                AWS (Conta AMEI)                                  │
│                                                                                  │
│   ┌─────────────────────────────┐    ┌─────────────────────────────────────┐   │
│   │                             │    │                                     │   │
│   │       AWS COGNITO           │    │      SECRETS MANAGER                │   │
│   │                             │    │                                     │   │
│   │  User Pool:                 │    │  webdental/cognito/admin-credentials│   │
│   │  us-east-1_F48JuTtz8        │    │                                     │   │
│   │                             │    │  Rotação: 15 dias                   │   │
│   │                             │    │                                     │   │
│   └─────────────────────────────┘    └─────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Estrutura de Arquivos

```
app/SSO/
├── Controllers/
│   ├── TokenExchangeController.php
│   ├── RefreshTokenController.php
│   └── LogoutController.php
│
├── Services/
│   ├── Contracts/
│   │   ├── UserSessionServiceInterface.php
│   │   ├── CognitoAuthServiceInterface.php
│   │   └── CognitoAdminServiceInterface.php
│   │
│   ├── UserSessionService.php
│   ├── CognitoAuthService.php
│   ├── CognitoAdminService.php
│   ├── AwsCredentialsService.php
│   └── UserResolverService.php
│
├── Repositories/
│   ├── Contracts/
│   │   ├── SessionRepositoryInterface.php
│   │   └── RefreshTokenRepositoryInterface.php
│   │
│   ├── ValkeySessionRepository.php
│   └── ValkeyRefreshTokenRepository.php
│
├── Http/
│   └── Middleware/
│       └── AuthenticateWithValkey.php
│
├── DTO/
│   ├── UserSessionDataDTO.php
│   └── RefreshTokenDataDTO.php
│
├── Exceptions/
│   ├── CognitoAdminException.php
│   └── SessionCreationFailedException.php
│
├── Helpers/
│   └── NanoIdGenerator.php
│
├── Models/
│   └── AuthenticatedUser.php
│
└── Providers/
    └── SSOServiceProvider.php
```

## Dependências entre Componentes

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  AuthenticateWithValkey ──────► UserSessionService              │
│                                        │                        │
│                                        ├──► SessionRepository   │
│                                        │                        │
│                                        ├──► RefreshTokenRepo    │
│                                        │                        │
│                                        ├──► CognitoAuthService  │
│                                        │                        │
│                                        └──► CognitoAdminService │
│                                                   │             │
│                                                   ▼             │
│                                        AwsCredentialsService    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
