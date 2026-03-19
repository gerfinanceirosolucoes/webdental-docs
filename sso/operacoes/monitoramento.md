# Monitoramento

## Métricas Importantes

### Taxa de Sucesso de Login

| Métrica | Bom | Atenção | Crítico |
|---------|-----|---------|---------|
| Taxa de sucesso | > 98% | 95-98% | < 95% |
| Latência média | < 500ms | 500ms-1s | > 1s |
| Erros por hora | < 10 | 10-50 | > 50 |

### Sessões

| Métrica | Como medir |
|---------|------------|
| Sessões ativas | `KEYS webdental:session:*` |
| Refresh tokens ativos | `KEYS webdental:refresh:*` |
| Sessões por usuário | `SCARD webdental:user_sessions:{sub}` |

### Valkey

| Métrica | Bom | Atenção | Crítico |
|---------|-----|---------|---------|
| Memória usada | < 70% | 70-85% | > 85% |
| Conexões ativas | < 80% max | 80-90% | > 90% |
| Latência GET | < 1ms | 1-5ms | > 5ms |

---

## Comandos de Monitoramento

### Valkey

```bash
# Conectar ao Valkey
docker exec -it valkey valkey-cli

# Ou via telnet/redis-cli
redis-cli -h localhost -p 6379
```

```bash
# Estatísticas gerais
INFO

# Memória
INFO memory

# Clientes conectados
INFO clients

# Número de chaves
DBSIZE

# Listar sessões (cuidado em produção!)
KEYS webdental:session:*

# Contar sessões
EVAL "return #redis.call('keys', 'webdental:session:*')" 0

# Ver uma sessão específica
GET webdental:session:V1StGXR8_Z5jdHi6B-myT

# TTL de uma sessão
TTL webdental:session:V1StGXR8_Z5jdHi6B-myT

# Monitorar comandos em tempo real
MONITOR
```

### Laravel Logs

```bash
# Logs em tempo real
tail -f storage/logs/laravel.log

# Filtrar por SSO
tail -f storage/logs/laravel.log | grep -E "(SSO|Cognito|Session)"

# Últimos erros
grep -i error storage/logs/laravel.log | tail -50

# Erros de hoje
grep "$(date +%Y-%m-%d)" storage/logs/laravel.log | grep -i error
```

### AWS CloudWatch

```bash
# Métricas do Cognito (via AWS CLI)
aws cloudwatch get-metric-statistics \
    --namespace AWS/Cognito \
    --metric-name SignInSuccesses \
    --dimensions Name=UserPoolId,Value=us-east-1_F48JuTtz8 \
    --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 \
    --statistics Sum
```

---

## Alertas Recomendados

### Críticos (notificação imediata)

| Alerta | Condição | Ação |
|--------|----------|------|
| Valkey indisponível | Conexão falha | Verificar container/serviço |
| Taxa de erro > 10% | Erros/total > 0.1 | Investigar logs |
| Cognito timeout | Latência > 5s | Verificar rede AWS |

### Atenção (notificação durante horário comercial)

| Alerta | Condição | Ação |
|--------|----------|------|
| Memória Valkey > 80% | Usado/Total > 0.8 | Planejar scale ou limpeza |
| Taxa de refresh alta | > 100/hora | Verificar TTL do access token |
| Sessões órfãs | Sessões sem TTL | Limpar manualmente |

---

## Dashboard Sugerido

### Gráficos

1. **Logins por hora** - Sucesso vs Falha
2. **Latência de autenticação** - P50, P95, P99
3. **Sessões ativas** - Total e por hora
4. **Uso de memória Valkey** - Trend 24h
5. **Erros por tipo** - Agrupado por código de erro

### Métricas em Tempo Real

```
┌─────────────────────────────────────────────────────────────┐
│                    SSO Dashboard                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Sessões Ativas: 1,234          Logins/hora: 45             │
│  Refresh/hora: 12               Erros/hora: 2               │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Logins últimas 24h                                  │   │
│  │ ████████████████████████████░░░░░░░░░░░░░░░░░░░░░░ │   │
│  │ 00  04  08  12  16  20  00                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Últimos Erros:                                             │
│  • 14:32 - USER_NOT_FOUND - user@email.com                 │
│  • 14:28 - REFRESH_FAILED - token expirado                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Logs Estruturados

Para facilitar o monitoramento, os logs do SSO seguem um formato estruturado:

```php
// Exemplo de log
Log::info('SSO: Login success', [
    'action' => 'login',
    'email' => $email,
    'session_id' => $sessionId,
    'duration_ms' => $duration,
]);

Log::error('SSO: Login failed', [
    'action' => 'login',
    'email' => $email,
    'error' => $errorCode,
    'message' => $errorMessage,
]);
```

### Campos Padrão

| Campo | Descrição |
|-------|-----------|
| `action` | Operação (login, logout, refresh, etc) |
| `email` | Email do usuário (quando disponível) |
| `session_id` | ID da sessão (quando disponível) |
| `duration_ms` | Duração da operação em ms |
| `error` | Código de erro (quando aplicável) |

---

## Troubleshooting via Monitoramento

### Muitos erros USER_NOT_FOUND

**Causa provável:** Usuários existem no Cognito mas não no Webdental.

**Ação:** Verificar sincronização de usuários entre sistemas.

### Alta taxa de refresh

**Causa provável:** Access token com TTL muito curto.

**Ação:** Verificar configuração do Cognito App Client.

### Sessões não expiram

**Causa provável:** TTL não está sendo setado corretamente.

**Ação:**
```bash
# Verificar TTL de algumas sessões
redis-cli TTL webdental:session:xxx
# Deve retornar número positivo, não -1
```

### Memória do Valkey crescendo

**Causa provável:** Sessões não estão expirando.

**Ação:**
```bash
# Verificar chaves sem TTL
redis-cli --scan --pattern 'webdental:*' | while read key; do
    ttl=$(redis-cli TTL "$key")
    if [ "$ttl" = "-1" ]; then
        echo "$key has no TTL"
    fi
done
```
