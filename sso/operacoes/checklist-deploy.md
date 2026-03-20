# Checklist de Deploy

## Pré-Deploy

### Código

- [ ] Branch `master` atualizada
- [ ] Pull request aprovado
- [ ] Testes unitários passando (`./vendor/bin/phpunit`)
- [ ] Sem conflitos de merge

### Configuração AWS (uma vez)

- [ ] IAM User criado (conta AMEI)
- [ ] Permissões Cognito configuradas (conta AMEI)
- [ ] Secret criado no Secrets Manager (conta AMEI)
- [ ] Rotação de credenciais configurada (15 dias) (conta AMEI)

### GitHub Secrets (uma vez)

- [ ] `AWS_COGNITO_ACCESS_KEY_ID` configurado
- [ ] `AWS_COGNITO_SECRET_ACCESS_KEY` configurado
- [ ] `AWS_COGNITO_CREDENTIALS_SECRET_ID` configurado

### Infraestrutura

- [ ] Valkey acessível
- [ ] Cognito User Pool configurado (conta AMEI)
- [ ] Certificados SSL válidos

---

## Durante o Deploy

O workflow do GitHub Actions executa automaticamente:

1. ✅ Backup da versão atual
2. ✅ Pull do código mais recente
3. ✅ Instalação de dependências
4. ✅ Injeção de variáveis AWS
5. ✅ Limpeza de caches
6. ✅ Ajuste de permissões

### Monitorar

```bash
# Acompanhar logs durante deploy
tail -f storage/logs/laravel.log
```

---

## Pós-Deploy

### Verificações Obrigatórias

- [ ] **Login funciona** - Testar fluxo completo de login
- [ ] **Logout funciona** - Verificar se remove sessão e cookie
- [ ] **Refresh funciona** - Esperar access token expirar e verificar renovação
- [ ] **Cross-subdomain** - Testar acesso entre webdental e ng.webdental

### Verificações Técnicas

```bash
# SSH no servidor

# 1. Verificar variáveis AWS no .env
grep AWS_COGNITO .env

# 2. Testar conexão com Valkey
php artisan tinker
>>> app('redis')->ping();

# 3. Testar conexão com Cognito
>>> app(\App\SSO\Services\Contracts\CognitoAdminServiceInterface::class)->adminGetUser('email@teste.com');

# 4. Verificar logs por erros
tail -100 storage/logs/laravel.log | grep -i error
```

### Verificações via Browser

1. Abrir DevTools → Network
2. Fazer login
3. Verificar:
   - [ ] Request para `/token/exchange` retorna 200
   - [ ] Cookie `webdental_session_id` foi setado
   - [ ] Cookie tem atributos corretos (Secure, HttpOnly, SameSite=None)
4. Fazer logout
5. Verificar:
   - [ ] Cookie foi removido (Max-Age=0)

---

## Rollback

Se algo der errado:

### Via GitHub Actions

1. Acesse a aba Actions
2. Encontre o último deploy bem-sucedido
3. Clique em "Re-run jobs"

### Via SSH

```bash
# Listar backups
ls -la /home/sistemas/www/webdental_novo/ | grep API_autobackup

# Restaurar
cd /home/sistemas/www/webdental_novo/
sudo rm -rf API
sudo mv API_autobackup_YYYYMMDD_HHMMSS API

# Limpar cache
cd API
php artisan config:clear
php artisan cache:clear
```

---

## Contatos de Emergência

| Problema | Contato |
|----------|---------|
| Cognito/AWS AMEI | daniel.rauh@amorsaude.com |
| Servidor/Valkey | stephano.ferreira@amorsaude.com |
| Aplicação | stephano.ferreira@amorsaude.com |

---
