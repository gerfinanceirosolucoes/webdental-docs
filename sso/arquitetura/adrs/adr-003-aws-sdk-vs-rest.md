# ADR-003: AWS SDK vs REST para Cognito Admin

## Status

✅ Aceito

## Data

Março 2026

## Contexto

O `UserSessionService` precisava fazer `GlobalSignOut` no Cognito para invalidar todos os refresh tokens do usuário durante o logout. A implementação inicial usava chamadas REST diretas com Guzzle:

```php
// Implementação anterior
$client = new Client(['base_uri' => $cognitoEndpoint]);
$client->post('/oauth2/revoke', [
    'form_params' => [
        'token' => $refreshToken,
        'client_id' => $clientId,
    ],
]);
```

Problemas:
1. **Revoke vs GlobalSignOut**: O endpoint `/oauth2/revoke` revoga apenas um token específico, não todos os tokens do usuário
2. **Token expirado**: Se o access_token expirou, não conseguimos fazer revoke
3. **Código espalhado**: Chamadas REST em vários lugares, difícil manter

Além disso, surgiram novas necessidades:
- Criar usuários no Cognito (migração)
- Desativar/ativar usuários
- Atualizar atributos
- Definir senhas administrativamente

## Decisão

Usar o **AWS SDK for PHP** para operações administrativas do Cognito, encapsulado no `CognitoAdminService`.

### Implementação

```php
// Novo: usando AWS SDK
class CognitoAdminService implements CognitoAdminServiceInterface
{
    private $client; // CognitoIdentityProviderClient

    public function globalSignOut(string $accessToken): bool
    {
        $this->client->globalSignOut([
            'AccessToken' => $accessToken,
        ]);
    }

    public function adminUserGlobalSignOut(string $username): bool
    {
        $this->client->adminUserGlobalSignOut([
            'UserPoolId' => $this->userPoolId,
            'Username' => $username,
        ]);
    }
}
```

### Estratégia de GlobalSignOut

```php
// No UserSessionService
private function globalSignOutCognito($accessToken, $sub)
{
    // Tenta primeiro com accessToken (não requer IAM)
    if ($this->cognitoAdminService->globalSignOut($accessToken)) {
        return;
    }

    // Fallback: usa adminUserGlobalSignOut (requer IAM, funciona com token expirado)
    $this->cognitoAdminService->adminUserGlobalSignOut($sub);
}
```

## Consequências

### Positivas

- **GlobalSignOut correto**: Invalida TODOS os refresh tokens do usuário
- **Funciona com token expirado**: `adminUserGlobalSignOut` usa IAM, não precisa de access_token
- **Código centralizado**: Um service para todas as operações Cognito
- **Novos casos de uso**: Criar usuários, desativar, atualizar atributos
- **Melhor tratamento de erros**: SDK tem exceções tipadas
- **Retry automático**: SDK faz retry em erros temporários

### Negativas

- **Dependência adicional**: `aws/aws-sdk-php` (~30MB)
- **Credenciais IAM**: Precisa de IAM user com permissões específicas
- **Complexidade de credenciais**: Credenciais rotacionadas no Secrets Manager

## Alternativas Consideradas

### Manter REST com Guzzle

**Rejeitado porque:**
- `/oauth2/revoke` não faz GlobalSignOut real
- Não funciona com token expirado
- Menos operações disponíveis

### Lambda intermediária

**Rejeitado porque:**
- Complexidade adicional
- Latência extra
- Mais pontos de falha

## Credenciais

As credenciais IAM são gerenciadas pelo Secrets Manager com rotação automática a cada 15 dias:

```
┌─────────────────┐     ┌─────────────────────┐     ┌─────────────┐
│                 │     │                     │     │             │
│   Laravel API   │────►│   Secrets Manager   │────►│   Cognito   │
│                 │     │   (credenciais)     │     │             │
└─────────────────┘     └─────────────────────┘     └─────────────┘

Fluxo:
1. API busca credenciais no Secrets Manager
2. Credenciais rotacionadas são retornadas
3. API usa credenciais para chamar Cognito
```

## Arquivos criados

| Arquivo | Descrição |
|---------|-----------|
| `CognitoAdminService.php` | Implementação do serviço |
| `CognitoAdminServiceInterface.php` | Contrato |
| `CognitoAdminException.php` | Exceções tipadas |
| `AwsCredentialsService.php` | Busca credenciais do Secrets Manager |
