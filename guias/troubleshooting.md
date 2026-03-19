# Troubleshooting

Guia para diagnóstico e resolução de problemas comuns do SSO.

## Problemas de Login

### Usuário não consegue fazer login

**Sintomas:**
- Tela de login do Cognito aparece mas não redireciona após autenticar
- Erro "Invalid redirect_uri"

**Diagnóstico:**
1. Verificar se a URL de callback está configurada no Cognito
2. Verificar se `COGNITO_REDIRECT_URI` no `.env` bate com o Cognito

**Solução:**
```bash
# Verificar configuração
grep COGNITO_REDIRECT_URI .env

# No console do Cognito:
# App client settings → Callback URL(s)
```

---

### Redirect loop após login

**Sintomas:**
- Usuário faz login no Cognito
- Redireciona de volta para o Cognito infinitamente

**Diagnóstico:**
1. Cookie não está sendo setado
2. Sessão não está sendo criada no Valkey

**Solução:**
```bash
# Verificar se Valkey está acessível
php artisan tinker
>>> app('redis')->ping();

# Verificar logs
tail -f storage/logs/laravel.log | grep -i session

# Verificar se cookie está sendo setado
# No navegador: DevTools → Application → Cookies
```

---

### Erro "USER_NOT_FOUND"

**Sintomas:**
- Login funciona no Cognito
- Redireciona para `/sso/user-not-found.php`

**Causa:**
Usuário existe no Cognito mas não no banco local do Webdental.

**Solução:**
1. Verificar se o e-mail do Cognito existe na tabela `tbUsuarios`
2. Criar o usuário no Webdental se necessário

```sql
SELECT * FROM tbUsuarios WHERE ds_email = 'email@exemplo.com';
```

---

## Problemas de Sessão

### Erro 401 em todas as requisições

**Sintomas:**
- Usuário está logado mas recebe 401 em chamadas à API
- `sso_expired: true` no response

**Diagnóstico:**
```bash
# Verificar se sessão existe no Valkey
docker exec -it valkey valkey-cli
> KEYS webdental:session:*
> GET webdental:session:{session_id}
```

**Soluções possíveis:**

1. **Sessão expirou:** Verificar se o TTL está correto
2. **Cookie não enviado:** Verificar se `withCredentials: true` no Angular
3. **CORS bloqueando:** Verificar `SSO_ALLOWED_ORIGINS`

---

### Sessão expira muito rápido

**Sintomas:**
- Usuário é deslogado após poucos minutos
- Precisa fazer login frequentemente

**Diagnóstico:**
```bash
# Verificar TTL da sessão
docker exec -it valkey valkey-cli
> TTL webdental:session:{session_id}
```

**Solução:**
O TTL é baseado no `access_token` do Cognito (geralmente 1 hora). Para sessões mais longas, o refresh deve estar funcionando.

Verificar se o refresh está configurado corretamente no Angular.

---

### Logout não funciona em todos os sistemas

**Sintomas:**
- Usuário faz logout no Webdental
- Continua logado no Angular ou vice-versa

**Diagnóstico:**
1. Verificar se `GlobalSignOut` está sendo chamado
2. Verificar se o cookie está sendo removido

```bash
# Verificar logs
grep "GlobalSignOut" storage/logs/laravel.log
```

**Solução:**
Verificar se o domínio do cookie está correto (`.webdental.com.br` para produção).

---

## Problemas de Conexão

### Erro de conexão com Valkey

**Sintomas:**
- `ConnectionException: Connection refused`
- Timeout nas requisições

**Diagnóstico:**
```bash
# Testar conectividade
telnet {VALKEY_HOST} {VALKEY_PORT}

# Verificar se Valkey está rodando
docker ps | grep valkey

# Testar via CLI
docker exec -it valkey valkey-cli ping
```

**Solução:**
1. Verificar se o container/serviço está rodando
2. Verificar configuração de rede/firewall
3. Verificar variáveis `VALKEY_HOST`, `VALKEY_PORT`, `VALKEY_PASSWORD`

---

### Erro de conexão com Cognito

**Sintomas:**
- Timeout ao trocar authorization code
- Erro ao fazer refresh de tokens

**Diagnóstico:**
```bash
# Testar conectividade com Cognito
curl -I https://cognito-idp.us-east-1.amazonaws.com

# Verificar logs
grep "Cognito" storage/logs/laravel.log
```

**Solução:**
1. Verificar conectividade de rede
2. Verificar se as credenciais IAM são válidas
3. Verificar se o User Pool ID está correto

---

### Erro de credenciais AWS

**Sintomas:**
- `CredentialsException`
- `InvalidSignatureException`

**Diagnóstico:**
```bash
# Verificar se as variáveis estão no .env
grep AWS_COGNITO .env

# Testar credenciais
aws sts get-caller-identity --profile cognito-secrets
```

**Solução:**
1. Verificar se as credenciais estão corretas
2. Verificar se o IAM User tem as permissões necessárias
3. Verificar se as credenciais não expiraram

---

## Problemas de CORS

### Erro "CORS policy: No 'Access-Control-Allow-Origin'"

**Sintomas:**
- Requisições do Angular falham com erro de CORS
- Funciona via Postman mas não no browser

**Diagnóstico:**
```bash
# Verificar configuração
grep SSO_ALLOWED_ORIGINS .env
```

**Solução:**
1. Adicionar a origem à lista de permitidas
2. Verificar se não há espaços extras na configuração
3. Reiniciar o servidor após alterar

---

### Cookies não enviados em requisições cross-origin

**Sintomas:**
- Cookie existe no browser
- Não é enviado para a API

**Diagnóstico:**
1. Verificar se `withCredentials: true` no Angular
2. Verificar se `SameSite=None; Secure` no cookie
3. Verificar se HTTPS está ativo

**Solução Angular:**
```typescript
this.http.get(url, { withCredentials: true });
```

---

## Ferramentas de Diagnóstico

### Logs

```bash
# Laravel logs
tail -f storage/logs/laravel.log

# Filtrar por SSO
tail -f storage/logs/laravel.log | grep -E "(SSO|Cognito|Session)"
```

### Valkey

```bash
# Listar todas as sessões
docker exec -it valkey valkey-cli KEYS "webdental:session:*"

# Ver dados de uma sessão
docker exec -it valkey valkey-cli GET "webdental:session:{id}"

# Ver TTL
docker exec -it valkey valkey-cli TTL "webdental:session:{id}"

# Monitorar em tempo real
docker exec -it valkey valkey-cli MONITOR
```

### Tinker

```php
// Testar conexão com Valkey
app('redis')->ping();

// Testar CognitoAdminService
$admin = app(\App\SSO\Services\Contracts\CognitoAdminServiceInterface::class);
$admin->adminGetUser('email@teste.com');

// Buscar sessão
$session = app(\App\SSO\Services\Contracts\UserSessionServiceInterface::class);
$session->getSession('nano_id_aqui');
```

### DevTools do Browser

1. **Network tab:** Verificar se cookies estão sendo enviados
2. **Application tab → Cookies:** Verificar se cookie existe e atributos
3. **Console:** Verificar erros de CORS ou JavaScript
