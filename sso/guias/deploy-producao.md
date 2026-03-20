# Deploy em Produção

Este guia documenta todos os passos necessários para fazer o deploy do SSO em produção.

## Pré-requisitos

Antes de iniciar, certifique-se de que:

- [ ] Cognito User Pool de produção está configurado
- [ ] Valkey de produção está disponível
- [ ] Credenciais IAM foram geradas
- [ ] GitHub Secrets foram configurados

## 1. Configuração do GitHub Secrets

Acesse: **Repository → Settings → Secrets and variables → Actions**

| Secret | Descrição | Quem fornece |
|--------|-----------|--------------|
| `AWS_COGNITO_ACCESS_KEY_ID` | Access Key do IAM User | DevOps AMEI |
| `AWS_COGNITO_SECRET_ACCESS_KEY` | Secret Key do IAM User | DevOps AMEI |
| `AWS_COGNITO_CREDENTIALS_SECRET_ID` | ID do secret no Secrets Manager | DevOps AMEI |

## 2. Variáveis de Ambiente (Produção)

As seguintes variáveis devem estar no `.env` de produção:

### Cognito

```env
COGNITO_REGION=us-east-1
COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
COGNITO_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxx
COGNITO_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxx
COGNITO_DOMAIN=https://seu-dominio.auth.us-east-1.amazoncognito.com
COGNITO_REDIRECT_URI=https://sistema.webdentalsolucoes.io/sso/callback.php
COGNITO_LOGOUT_URI=https://sistema.webdentalsolucoes.io/
COGNITO_SCOPES=openid,email,profile
```

### Valkey

```env
VALKEY_HOST=seu-valkey-host
VALKEY_PORT=6379
VALKEY_SCHEME=tls
VALKEY_PASSWORD=sua-senha
VALKEY_DATABASE=0
VALKEY_SESSION_PREFIX=webdental:session:
VALKEY_USER_SESSION_PREFIX=webdental:user_sessions:
VALKEY_REFRESH_TTL=28800
```

### Cookie

```env
SSO_COOKIE_NAME=webdental_session_id
SSO_COOKIE_DOMAIN=.webdental.com.br
SSO_COOKIE_SECURE=true
```

### CORS

```env
SSO_ALLOWED_ORIGINS=https://sistema.webdentalsolucoes.io,https://ng.webdentalsolucoes.io,https://novo.webdentalsolucoes.io,https://api.webdentalsolucoes.io,https://apiwebvidas.webdentalsolucoes.io
```

## 3. Workflow do GitHub Actions

O workflow injeta automaticamente as credenciais AWS durante o deploy:

```yaml
# Trecho do workflow
- name: Configure AWS Cognito credentials
  run: |
    sudo sed -i '/^AWS_COGNITO_ACCESS_KEY_ID=/d' .env
    sudo sed -i '/^AWS_COGNITO_SECRET_ACCESS_KEY=/d' .env
    sudo sed -i '/^AWS_COGNITO_CREDENTIALS_SECRET_ID=/d' .env
    
    echo "AWS_COGNITO_ACCESS_KEY_ID=${{ secrets.AWS_COGNITO_ACCESS_KEY_ID }}" | sudo tee -a .env
    echo "AWS_COGNITO_SECRET_ACCESS_KEY=${{ secrets.AWS_COGNITO_SECRET_ACCESS_KEY }}" | sudo tee -a .env
    echo "AWS_COGNITO_CREDENTIALS_SECRET_ID=${{ secrets.AWS_COGNITO_CREDENTIALS_SECRET_ID }}" | sudo tee -a .env
```

## 4. Checklist de Deploy

### Antes do Deploy

- [ ] Testes unitários passando (`./vendor/bin/phpunit`)
- [ ] Código revisado e aprovado
- [ ] GitHub Secrets configurados
- [ ] Cognito de produção configurado
- [ ] Valkey de produção acessível

### Durante o Deploy

O workflow executa automaticamente:
1. Backup da versão atual
2. Pull do código mais recente
3. Instalação de dependências
4. Injeção de variáveis AWS
5. Limpeza de caches
6. Ajuste de permissões

### Após o Deploy

- [ ] Testar login em produção
- [ ] Testar logout em produção
- [ ] Verificar logs por erros
- [ ] Testar acesso entre subdomínios
- [ ] Validar que sessões antigas continuam funcionando

## 5. Verificação Pós-Deploy

### Testar via SSH

```bash
ssh user@servidor

# Verificar se as variáveis estão no .env
grep AWS_COGNITO .env

# Testar conexão com Cognito
php artisan tinker
```

```php
$admin = app(\App\SSO\Services\Contracts\CognitoAdminServiceInterface::class);
$user = $admin->adminGetUser('email-de-teste@empresa.com');
print_r($user);
```

### Verificar logs

```bash
tail -f storage/logs/laravel.log | grep -i sso
tail -f storage/logs/laravel.log | grep -i cognito
```

## 6. Rollback

Se algo der errado, o workflow cria backup automático:

```bash
# No servidor
cd /home/sistemas/www/webdental_novo

# Listar backups
ls -la | grep API_autobackup

# Restaurar backup
sudo rm -rf API
sudo mv API_autobackup_YYYYMMDD_HHMMSS API
```

## 7. Monitoramento

### Métricas a observar

| Métrica | Onde verificar |
|---------|----------------|
| Erros de autenticação | CloudWatch Logs / Laravel Logs |
| Latência de login | CloudWatch Metrics |
| Taxa de refresh | Logs do Valkey |
| Sessões ativas | Valkey (`KEYS webdental:session:*`) |

### Alertas recomendados

- [ ] Taxa de erro de login > 5%
- [ ] Latência de token/exchange > 2s
- [ ] Erros de conexão com Valkey
- [ ] Erros de conexão com Cognito

## 8. Contatos

| Responsável | Área | Contato |
|-------------|------|---------|
| DevOps AMEI | Cognito, Secrets Manager | daniel.rauh@amorsaude.com |
| DevOps Webdental | EC2, Valkey | [email] |
| Desenvolvimento | Aplicação | stephano.ferreira@amorsaude.com |
