# ADR-001: Cookie-based Sessions

## Status

✅ Aceito

## Data

Fevereiro 2026

## Contexto

O Webdental possui múltiplas aplicações em diferentes subdomínios que precisam compartilhar o estado de autenticação:

- `webdental.local` (PHP Legado)
- `ng.webdental.local` (Angular)
- `angularjs.webdental.local` (AngularJS)
- `api.webdental.local` (Laravel API)

Precisamos de um mecanismo que:
1. Funcione em todas as stacks (PHP, Angular, AngularJS)
2. Seja seguro contra XSS
3. Permita logout global
4. Não exija mudanças significativas no código legado

## Decisão

Usar **cookies HTTPOnly** para armazenar o session_id, com as seguintes configurações:

```php
[
    'name' => 'webdental_session_id',
    'domain' => '.webdental.local',  // Compartilhado entre subdomínios
    'secure' => true,                 // Apenas HTTPS
    'httpOnly' => true,               // Não acessível via JavaScript
    'sameSite' => 'None',             // Permite cross-origin (necessário para subdomínios)
    'path' => '/',
]
```

O cookie contém apenas um **NanoID de 21 caracteres** que referencia a sessão armazenada no Valkey. Nenhum dado sensível é armazenado no cookie.

## Consequências

### Positivas

- **Segurança contra XSS**: Tokens nunca são expostos ao JavaScript
- **Compatibilidade**: Funciona nativamente em todas as stacks sem mudanças
- **Simplicidade**: Browser gerencia automaticamente o envio do cookie
- **Logout automático**: Ao fechar o browser, sessão expira (session cookie)

### Negativas

- **Requer HTTPS**: `SameSite=None` exige `Secure=true`
- **Configuração de CORS**: Necessário `credentials: true` nas requisições
- **Complexidade em dev local**: Precisa de certificados SSL locais (mkcert)

## Alternativas Consideradas

### Bearer Token em Header

```javascript
headers: { 'Authorization': 'Bearer ' + token }
```

**Rejeitado porque:**
- Token precisa ser armazenado no localStorage (vulnerável a XSS)
- Código legado PHP precisaria de mudanças significativas
- Angular e AngularJS precisariam de interceptors diferentes

### Session Storage

**Rejeitado porque:**
- Não compartilha entre abas do browser
- Não funciona entre subdomínios
- Vulnerável a XSS

### JWT em Cookie

**Rejeitado porque:**
- JWT é grande (aumenta tamanho de cada request)
- Não podemos invalidar JWT antes da expiração
- Complexidade adicional desnecessária
