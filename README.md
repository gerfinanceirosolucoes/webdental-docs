# Webdental Docs

Documentação técnica do ecossistema Webdental.

## Projetos Documentados

| Projeto | Descrição | Status |
|---------|-----------|--------|
| [SSO](sso/README.md) | Sistema de autenticação unificada com AWS Cognito | 📝 Documentado |
| API | API Laravel do Webdental | ⏳ Em breve |
| Angular | Frontend Angular 6 | ⏳ Em breve |
| AngularJS | Frontend legado | ⏳ Em breve |
| Infraestrutura | AWS, CI/CD, Monitoramento | ⏳ Em breve |

## Stack Técnica

| Componente | Tecnologia |
|------------|------------|
| Backend | Laravel 5.4 / PHP 7.0 |
| Frontend | Angular 6, AngularJS |
| Banco de Dados | MySQL / MariaDB |
| Cache/Sessões | Valkey (Redis) |
| Autenticação | AWS Cognito |
| Infraestrutura | AWS (EC2, RDS, ECS, MSK) |

## Como Contribuir

1. Clone o repositório
2. Crie uma branch (`git checkout -b docs/minha-alteracao`)
3. Faça suas alterações
4. Commit (`git commit -m "docs: descrição da alteração"`)
5. Push (`git push origin docs/minha-alteracao`)
6. Abra um Pull Request

## Convenções

- Arquivos em português
- Markdown (.md)
- Diagramas em ASCII ou Mermaid
- ADRs para decisões de arquitetura
