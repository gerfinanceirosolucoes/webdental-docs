# Sumário

## Webdental

* [Visão Geral](README.md)

## SSO

* [Introdução](sso/README.md)
* [Glossário](sso/introducao/glossario.md)

### Arquitetura

* [Visão Geral da Arquitetura](sso/arquitetura/visao-geral.md)
* [Decisões de Design (ADRs)](sso/arquitetura/adrs/README.md)
  * [ADR-001: Cookie-based Sessions](sso/arquitetura/adrs/adr-001-cookie-based-sessions.md)
  * [ADR-002: Valkey para Sessões](sso/arquitetura/adrs/adr-002-valkey-sessoes.md)
  * [ADR-003: AWS SDK vs REST](sso/arquitetura/adrs/adr-003-aws-sdk-vs-rest.md)
* [Fluxos de Autenticação](sso/arquitetura/fluxos-autenticacao.md)
* [Diagrama de Componentes](sso/arquitetura/diagrama-componentes.md)

### Guias

* [Configuração do Ambiente Local](sso/guias/configuracao-ambiente-local.md)
* [Deploy em Produção](sso/guias/deploy-producao.md)
* [Troubleshooting](sso/guias/troubleshooting.md)

### Referência Técnica

* [CognitoAdminService](sso/referencia/cognito-admin-service.md)
* [UserSessionService](sso/referencia/user-session-service.md)
* [Middleware de Autenticação](sso/referencia/middleware-autenticacao.md)
* [Configurações](sso/referencia/configuracoes.md)

### APIs

* [POST /api/v1/token/exchange](sso/apis/token-exchange.md)
* [POST /api/v1/auth/refresh](sso/apis/auth-refresh.md)
* [POST /api/v1/auth/logout](sso/apis/auth-logout.md)

### Operações

* [Checklist de Deploy](sso/operacoes/checklist-deploy.md)
* [Monitoramento](sso/operacoes/monitoramento.md)
* [Rotação de Credenciais](sso/operacoes/rotacao-credenciais.md)
