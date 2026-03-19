# Sumário

## Introdução

* [Visão Geral](README.md)
* [Glossário](introducao/glossario.md)

## Arquitetura

* [Visão Geral da Arquitetura](arquitetura/visao-geral.md)
* [Decisões de Design (ADRs)](arquitetura/adrs/README.md)
  * [ADR-001: Cookie-based Sessions](arquitetura/adrs/adr-001-cookie-based-sessions.md)
  * [ADR-002: Valkey para Sessões](arquitetura/adrs/adr-002-valkey-sessoes.md)
  * [ADR-003: AWS SDK vs REST](arquitetura/adrs/adr-003-aws-sdk-vs-rest.md)
* [Fluxos de Autenticação](arquitetura/fluxos-autenticacao.md)
* [Diagrama de Componentes](arquitetura/diagrama-componentes.md)

## Guias

* [Configuração do Ambiente Local](guias/configuracao-ambiente-local.md)
* [Deploy em Produção](guias/deploy-producao.md)
* [Troubleshooting](guias/troubleshooting.md)

## Referência Técnica

* [CognitoAdminService](referencia/cognito-admin-service.md)
* [UserSessionService](referencia/user-session-service.md)
* [Middleware de Autenticação](referencia/middleware-autenticacao.md)
* [Configurações (config/sso.php)](referencia/configuracoes.md)

## APIs

* [POST /api/v1/token/exchange](apis/token-exchange.md)
* [POST /api/v1/auth/refresh](apis/auth-refresh.md)
* [POST /api/v1/auth/logout](apis/auth-logout.md)

## Operações

* [Checklist de Deploy](operacoes/checklist-deploy.md)
* [Monitoramento](operacoes/monitoramento.md)
* [Rotação de Credenciais](operacoes/rotacao-credenciais.md)
