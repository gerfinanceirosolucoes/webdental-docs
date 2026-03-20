# Glossário

## Termos do SSO

### Access Token
Token JWT de curta duração (ttl configurável) emitido pelo Cognito. Usado para autorizar requisições à API. Contém informações sobre permissões do usuário.

### Cognito
Serviço da AWS que gerencia autenticação e autorização de usuários (no nosso contexto o Cognito é utilizado apenas para autenticação). Funciona como o "guardião" que valida quem é o usuário.

### Cookie HTTPOnly
Cookie que só pode ser acessado pelo servidor, não por JavaScript. Usado para armazenar o session_id (nanoID que é usado como chave do cache no Valkey e aponta para os tokens do Cognito e dados do prestador) de forma segura contra ataques XSS.

### GlobalSignOut
Operação do Cognito que invalida todos os refresh tokens de um usuário em todos os dispositivos. Usado para garantir logout completo.

### NanoID
Identificador único de 21 caracteres usado como session_id. Alternativa mais curta e segura ao UUID.

### Refresh Token
Token de longa duração (ttl configurável) usado para obter novos access tokens sem pedir login novamente. Armazenado de forma segura no Valkey.

### Session ID
Identificador único da sessão do usuário no Webdental. É um NanoID de 21 caracteres armazenado em cookie.

### SSO (Single Sign-On)
Sistema que permite ao usuário fazer login uma única vez e acessar múltiplas aplicações sem precisar autenticar novamente.

### Sub
Identificador único e imutável do usuário no Cognito. Formato UUID. Nunca muda, mesmo se o email mudar.

### TTL (Time To Live)
Tempo de vida de um dado antes de expirar automaticamente. No Valkey, as sessões têm TTL baseado na expiração do token.

### Valkey
Banco de dados em memória compatível com Redis. Usado para armazenar sessões com alta performance e TTL automático.

---

## Siglas

| Sigla | Significado |
|-------|-------------|
| ADR | Architecture Decision Record |
| API | Application Programming Interface |
| CORS | Cross-Origin Resource Sharing |
| IAM | Identity and Access Management (AWS) |
| JWT | JSON Web Token |
| MFA | Multi-Factor Authentication |
| OIDC | OpenID Connect |
| SPA | Single Page Application |
| SSO | Single Sign-On |
| TTL | Time To Live |

---

## Ambientes

| Ambiente | Domínio | Uso |
|----------|---------|-----|
| Local | `https://webdental.local` | Desenvolvimento |
| Staging | `https://sistema.homologacao.webdentalsolucoes.io` | Testes |
| Produção | `https://sistema.webdentalsolucoes.io` | Usuários reais |
