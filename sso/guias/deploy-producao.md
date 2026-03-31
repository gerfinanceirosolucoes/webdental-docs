# Deploy em Produção

Este guia documenta todos os passos necessários para fazer o deploy do SSO em produção.

## Pré-requisitos

Antes de iniciar, certifique-se de que:

* [ ] Cognito User Pool de produção está configurado
* [ ] Valkey de produção está disponível
* [ ] Credenciais IAM foram geradas
* [ ] GitHub Secrets foram configurados

## 1. Dependências do Servidor

**1.1 Extensão phpredis para PHP 7.0**

A extensão `phpredis` precisa estar compilada e habilitada para o PHP 7.0:

```
# Verificar se já está instalada
php7.0 -m | grep redis

# Se não estiver, compilar manualmente
cd /tmp
wget https://pecl.php.net/get/redis-5.3.7.tgz
tar xzf redis-5.3.7.tgz
cd redis-5.3.7
phpize7.0
./configure --with-php-config=/usr/bin/php-config7.0
make && sudo make install

# Habilitar o módulo
echo "extension=redis.so" | sudo tee /etc/php/7.0/mods-available/redis.ini
sudo phpenmod -v 7.0 redis
sudo systemctl restart php7.0-fpm

# Confirmar
php7.0 -m | grep redis
```

**1.2 Stunnel — Proxy TLS para Valkey**

O PHP 7.0 não consegue estabelecer conexão TLS diretamente com o ElastiCache Serverless. O stunnel atua como proxy local: o PHP conecta via TCP simples na porta 6381, e o stunnel faz o handshake TLS com o ElastiCache.

```
# Instalar
sudo apt-get install -y stunnel4

# Criar configuração
sudo tee /etc/stunnel/valkey.conf << 'EOF'
[valkey]
client = yes
accept = 127.0.0.1:6381
connect = SEU-ENDPOINT-VALKEY-PRODUCAO.serverless.sae1.cache.amazonaws.com:6379
verify = 2
CAfile = /etc/ssl/certs/ca-certificates.crt
EOF

# Habilitar no boot e iniciar
sudo sed -i 's/ENABLED=0/ENABLED=1/' /etc/default/stunnel4
sudo systemctl enable stunnel4
sudo systemctl start stunnel4

# Confirmar
sudo systemctl status stunnel4 | grep "Active"
```

> **Importante:** O stunnel deve ser iniciado antes do PHP-FPM. Verificar a ordem de boot após o deploy.

## 2. Configuração do GitHub Secrets

Acesse: **Repository → Settings → Secrets and variables → Actions**

| Secret                              | Descrição                       | Quem fornece |
| ----------------------------------- | ------------------------------- | ------------ |
| `AWS_COGNITO_ACCESS_KEY_ID`         | Access Key do IAM User          | DevOps AMEI  |
| `AWS_COGNITO_SECRET_ACCESS_KEY`     | Secret Key do IAM User          | DevOps AMEI  |
| `AWS_COGNITO_CREDENTIALS_SECRET_ID` | ID do secret no Secrets Manager | DevOps AMEI  |

## 3. Variáveis de Ambiente (Produção)

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
VALKEY_HOST=127.0.0.1
VALKEY_PORT=6381
VALKEY_SCHEME=tcp
VALKEY_PASSWORD=
VALKEY_DATABASE=0
VALKEY_SESSION_PREFIX=webdental:session:
VALKEY_USER_SESSION_PREFIX=webdental:user_sessions:
VALKEY_REFRESH_TTL=28800
```

> **Importante:** `VALKEY_SCHEME=tcp` e `VALKEY_PORT=6381` (porta do stunnel local). O endpoint real do ElastiCache fica apenas na configuração do stunnel (`/etc/stunnel/valkey.conf`).

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

## 4. Configuração do config/database.php

O bloco `valkey` deve estar dentro do array `redis`:

```
'redis' => [
    'client' => 'predis',
    'default' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD', null),
        'port' => env('REDIS_PORT', 6379),
        'database' => 0,
    ],
    'valkey' => [
        'scheme' => env('VALKEY_SCHEME', 'tcp'),
        'host' => env('VALKEY_HOST', '127.0.0.1'),
        'password' => env('VALKEY_PASSWORD', null),
        'port' => env('VALKEY_PORT', 6381),
        'database' => env('VALKEY_DATABASE', 0),
    ],
],
```

> **Nota:** Sem bloco `ssl` — o stunnel já gerencia o TLS transparentemente.

## 5. Workflow do GitHub Actions

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

## 6. Checklist de Deploy

### Antes do Deploy

* [ ] Testes unitários passando (`./vendor/bin/phpunit`)
* [ ] Código revisado e aprovado
* [ ] GitHub Secrets configurados
* [ ] Cognito de produção configurado
* [ ] phpredis compilado e habilitado para PHP 7.0
* [ ] stunnel instalado, configurado e habilitado no boot
* [ ] config/database.php com bloco valkey dentro do array redis
* [ ] Valkey de produção acessível

### Durante o Deploy

O workflow executa automaticamente:

1. Backup da versão atual
2. Pull do código mais recente
3. Instalação de dependências
4. Injeção de variáveis AWS
5. Limpeza de caches
6. Ajuste de permissões

### Após o Deploy

* [ ] Testar login em produção
* [ ] Testar logout em produção
* [ ] Verificar logs por erros
* [ ] Testar acesso entre subdomínios
* [ ] Validar que sessões antigas continuam funcionando
* [ ] Confirmar que stunnel está ativo: `sudo systemctl status stunnel4`
* [ ] Confirmar conexão Valkey: `redis-cli -p 6381 PING`

## 7. Verificação Pós-Deploy

### Testar via SSH

```bash
# Verificar stunnel
sudo systemctl status stunnel4

# Testar conexão Valkey via stunnel
redis-cli -p 6381 PING

# Verificar sessões ativas
redis-cli -p 6381 SCAN 0 MATCH "webdental:session:*" COUNT 100

# Verificar se variáveis estão no .env
grep "VALKEY\|AWS_COGNITO" .env
```

```php
# Testar conexão Cognito
php artisan tinker
>>> $admin = app(\App\SSO\Services\Contracts\CognitoAdminServiceInterface::class);
>>> $user = $admin->adminGetUser('email-de-teste@empresa.com');
>>> print_r($user);
```

### Verificar logs

```bash
tail -f storage/logs/laravel.log | grep -i "sso\|valkey\|cognito"
```

## 8. Rollback

Se algo der errado, o workflow cria backup automático:

```bash
# No servidor
cd /home/sistemas/www/webdental_novo

# Listar backups
ls -la | grep API_autobackup

# Restaurar backup
sudo rm -rf API
sudo mv API_autobackup_YYYYMMDD_HHMMSS API

# Após restaurar, reiniciar stunnel se necessário
sudo systemctl restart stunnel4
```

## 9. Monitoramento

### Métricas a observar

| Métrica               | Onde verificar                                        |
| --------------------- | ----------------------------------------------------- |
| Erros de autenticação | CloudWatch Logs / Laravel Logs                        |
| Latência de login     | CloudWatch Metrics                                    |
| Taxa de refresh       | Logs do Valkey                                        |
| Sessões ativas        | redis-cli -p 6381 SCAN 0 MATCH "webdental:session:\*" |
| Status do stunnel     | sudo systemctl status stunnel4                        |

### Alertas recomendados

* [ ] Taxa de erro de login > 5%
* [ ] Latência de token/exchange > 2s
* [ ] Erros de conexão com Valkey
* [ ] Erros de conexão com Cognito
* [ ] Stunnel inativo

## 10. Contatos

| Responsável      | Área                     | Contato                         |
| ---------------- | ------------------------ | ------------------------------- |
| DevOps AMEI      | Cognito, Secrets Manager | daniel.rauh@amorsaude.com       |
| DevOps Webdental | EC2, Valkey              | \[email]                        |
| Desenvolvimento  | Aplicação                | stephano.ferreira@amorsaude.com |
