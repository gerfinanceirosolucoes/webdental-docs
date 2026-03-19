# CognitoAdminService

## O que é

O `CognitoAdminService` é o serviço responsável por operações administrativas no AWS Cognito. Ele permite gerenciar usuários de forma programática, sem necessidade de acesso ao console da AWS.

## Quando usar

| Cenário | Método |
|---------|--------|
| Verificar se usuário existe no Cognito | `userExists()`, `adminGetUser()` |
| Criar usuário durante migração | `adminCreateUser()` |
| Desativar usuário que saiu da empresa | `adminDisableUser()` |
| Forçar logout de todas as sessões | `adminUserGlobalSignOut()` |
| Resetar senha administrativamente | `adminSetUserPassword()` |

## Como usar

### Injeção de Dependência

```php
use App\SSO\Services\Contracts\CognitoAdminServiceInterface;

class UserController extends Controller
{
    private $cognitoAdmin;

    public function __construct(CognitoAdminServiceInterface $cognitoAdmin)
    {
        $this->cognitoAdmin = $cognitoAdmin;
    }

    public function deactivate(string $email)
    {
        $this->cognitoAdmin->adminDisableUser($email);
    }
}
```

### Via Container

```php
$cognitoAdmin = app(CognitoAdminServiceInterface::class);
// ou
$cognitoAdmin = app('cognito.admin');
```

## Referência de Métodos

### adminGetUser

Busca informações de um usuário.

```php
$user = $cognitoAdmin->adminGetUser('user@example.com');
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$username` | string | Email ou sub do usuário |

**Retorno:**
```php
[
    'username' => 'user@example.com',
    'sub' => 'abc-123-def',
    'email' => 'user@example.com',
    'email_verified' => true,
    'status' => 'CONFIRMED',
    'enabled' => true,
    'created_at' => '2024-01-15 10:30:00',
    'updated_at' => '2024-03-10 14:20:00',
    'attributes' => [
        'sub' => 'abc-123-def',
        'email' => 'user@example.com',
        'custom:user_id' => '12345',
    ],
]
```

**Exceções:**
- `CognitoAdminException` com código `UserNotFoundException` se usuário não existe

---

### adminCreateUser

Cria um novo usuário no Cognito.

```php
// Simples
$user = $cognitoAdmin->adminCreateUser('new@example.com');

// Com atributos
$user = $cognitoAdmin->adminCreateUser(
    'new@example.com',
    [
        'custom:user_id' => '12345',
        'name' => 'João Silva',
    ],
    true // enviar convite por email
);
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$email` | string | Email do usuário |
| `$attributes` | array | Atributos adicionais (opcional) |
| `$sendInvite` | bool | Enviar email de convite (default: true) |

**Retorno:** Mesmo formato de `adminGetUser()`

**Exceções:**
- `CognitoAdminException` com código `UsernameExistsException` se usuário já existe

---

### adminDisableUser

Desativa um usuário. Ele não conseguirá fazer login.

```php
$success = $cognitoAdmin->adminDisableUser('user@example.com');
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$username` | string | Email ou sub do usuário |

**Retorno:** `bool` - true se sucesso

---

### adminEnableUser

Reativa um usuário previamente desativado.

```php
$success = $cognitoAdmin->adminEnableUser('user@example.com');
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$username` | string | Email ou sub do usuário |

**Retorno:** `bool` - true se sucesso

---

### adminUpdateUserAttributes

Atualiza atributos de um usuário.

```php
$success = $cognitoAdmin->adminUpdateUserAttributes(
    'user@example.com',
    [
        'custom:user_id' => '999',
        'name' => 'Novo Nome',
    ]
);
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$username` | string | Email ou sub do usuário |
| `$attributes` | array | Atributos a atualizar |

**Retorno:** `bool` - true se sucesso

---

### adminDeleteUser

Remove um usuário permanentemente.

```php
$success = $cognitoAdmin->adminDeleteUser('user@example.com');
```

⚠️ **Atenção:** Esta ação é irreversível.

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$username` | string | Email ou sub do usuário |

**Retorno:** `bool` - true se sucesso

---

### adminSetUserPassword

Define a senha de um usuário administrativamente.

```php
// Senha permanente
$success = $cognitoAdmin->adminSetUserPassword(
    'user@example.com',
    'NovaSenh@123',
    true
);

// Senha temporária (força troca no próximo login)
$success = $cognitoAdmin->adminSetUserPassword(
    'user@example.com',
    'TempPass123!',
    false
);
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$username` | string | Email ou sub do usuário |
| `$password` | string | Nova senha |
| `$permanent` | bool | Se true, não força troca (default: true) |

**Retorno:** `bool` - true se sucesso

---

### adminUserGlobalSignOut

Invalida todos os tokens de um usuário usando credenciais administrativas.

```php
$success = $cognitoAdmin->adminUserGlobalSignOut('user@example.com');
```

**Quando usar:**
- Forçar logout de todas as sessões
- Após mudança de permissões
- Quando não tem acesso ao access token

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$username` | string | Email ou sub do usuário |

**Retorno:** `bool` - true se sucesso

---

### globalSignOut

Invalida todos os tokens usando o access token do usuário.

```php
$success = $cognitoAdmin->globalSignOut($accessToken);
```

**Diferença de `adminUserGlobalSignOut`:**
- Não requer credenciais IAM
- Requer access token válido
- Retorna `false` se token expirado (não lança exceção)

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$accessToken` | string | Access token do usuário |

**Retorno:** `bool` - true se sucesso, false se token expirado

---

### listUsers

Lista usuários do User Pool.

```php
// Todos (até 60)
$users = $cognitoAdmin->listUsers();

// Filtrar por email
$users = $cognitoAdmin->listUsers(['email' => 'user@example.com']);

// Limitar resultados
$users = $cognitoAdmin->listUsers([], 10);
```

**Parâmetros:**
| Nome | Tipo | Descrição |
|------|------|-----------|
| `$filter` | array | Filtros (email, sub, status) |
| `$limit` | int | Máximo de resultados (max: 60) |

**Retorno:** Array de usuários no formato de `adminGetUser()`

---

### findUserByEmail

Atalho para buscar usuário por email.

```php
$user = $cognitoAdmin->findUserByEmail('user@example.com');
// Retorna null se não encontrar
```

---

### userExists

Verifica se um usuário existe.

```php
if ($cognitoAdmin->userExists('user@example.com')) {
    // usuário existe
}
```

**Retorno:** `bool`

---

## Tratamento de Erros

```php
use App\SSO\Exceptions\CognitoAdminException;

try {
    $user = $cognitoAdmin->adminGetUser('user@example.com');
} catch (CognitoAdminException $e) {
    if ($e->isUserNotFound()) {
        // Usuário não existe
    }

    if ($e->isRateLimited()) {
        // Muitas requisições, aguardar
    }

    // Informações do erro
    $code = $e->getErrorCode();      // 'UserNotFoundException'
    $status = $e->getHttpStatus();   // 404
    $message = $e->getMessage();     // 'User not found'
}
```

### Códigos de Erro

| Código | HTTP | Descrição |
|--------|------|-----------|
| `UserNotFoundException` | 404 | Usuário não encontrado |
| `UsernameExistsException` | 409 | Usuário já existe |
| `InvalidParameterException` | 400 | Parâmetro inválido |
| `InvalidPasswordException` | 400 | Senha não atende requisitos |
| `NotAuthorizedException` | 401 | Não autorizado |
| `TooManyRequestsException` | 429 | Rate limit excedido |

## Arquivos

| Arquivo | Localização |
|---------|-------------|
| Implementação | `app/SSO/Services/CognitoAdminService.php` |
| Interface | `app/SSO/Services/Contracts/CognitoAdminServiceInterface.php` |
| Exceção | `app/SSO/Exceptions/CognitoAdminException.php` |
| Testes | `tests/Unit/SSO/Services/CognitoAdminServiceTest.php` |
