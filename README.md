# SSO Webdental

## O que é

O SSO (Single Sign-On) do Webdental é o sistema de autenticação unificada que permite aos usuários acessarem múltiplas aplicações da empresa com um único login.

## Por que foi criado

Antes do SSO, cada sistema tinha seu próprio login:

| Sistema | Autenticação antiga |
|---------|---------------------|
| Webdental (PHP Legado) | Sessão PHP + banco de dados |
| Angular | Token próprio via API |
| AngularJS | Sessão compartilhada com PHP |
| AMEI | Sistema separado |

**Problemas:**
- Usuários precisavam fazer login múltiplas vezes
- Senhas diferentes para cada sistema
- Impossível saber se usuário estava logado em outro sistema
- Dificuldade de implementar logout global

## Como funciona

O SSO utiliza **AWS Cognito** como provedor de identidade centralizado e **Valkey** (Redis) para gerenciar sessões compartilhadas.

```
┌─────────────────────────────────────────────────────────────┐
│                         USUÁRIO                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      AWS COGNITO                            │
│                  (Provedor de Identidade)                   │
│         • Autenticação (usuário/senha ou Google)            │
│         • Tokens (ID, Access, Refresh)                      │
│         • Gerenciamento de usuários                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     API WEBDENTAL                           │
│                  (Laravel 5.4 / PHP 7.0)                    │
│         • Troca tokens Cognito por sessão interna           │
│         • Valida e renova sessões                           │
│         • Gerencia permissões                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        VALKEY                               │
│                   (Armazenamento de Sessões)                │
│         • Sessões com TTL automático                        │
│         • Compartilhado entre todas as stacks               │
│         • Suporte a logout global                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      APLICAÇÕES                             │
├─────────────────┬─────────────────┬─────────────────────────┤
│   Webdental     │     Angular     │       AngularJS         │
│   (PHP Legado)  │    (Angular 6)  │       (Legado)          │
└─────────────────┴─────────────────┴─────────────────────────┘
```

## Benefícios

| Benefício | Descrição |
|-----------|-----------|
| **Login único** | Usuário faz login uma vez e acessa todos os sistemas |
| **Logout global** | Ao sair de um sistema, sai de todos |
| **Segurança centralizada** | Políticas de senha, MFA, bloqueio de conta em um só lugar |
| **Integração futura com Google** | Preparado para login com Google Workspace |
| **Auditoria** | Logs centralizados de autenticação |

## Stack Técnica

| Componente | Tecnologia |
|------------|------------|
| Provedor de Identidade | AWS Cognito |
| Armazenamento de Sessões | Valkey (Redis-compatible) |
| API Backend | Laravel 5.4 / PHP 7.0 |
| Frontend Angular | Angular 6 |
| Frontend Legado | PHP + AngularJS |

## Status

| Fase | Status | Descrição |
|------|--------|-----------|
| Fase 1 | 🟡 Em desenvolvimento | Login com usuário/senha |
| Fase 2 | ⚪ Planejado | Login com Google Workspace |
| Fase 3 | ⚪ Planejado | Migração completa do login legado |

## Próximos Passos

1. Finalizar testes de integração
2. Deploy em ambiente de staging
3. Testes com usuários piloto
4. Rollout gradual em produção
