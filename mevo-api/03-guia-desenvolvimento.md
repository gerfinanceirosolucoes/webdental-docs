# Guia de Desenvolvimento — mevo-config.php

O backend PHP chama a Mevo API através do `mevo-proxy.php`, que lê endpoint e credencial do `mevo-config.php`. Este arquivo **não é versionado** — cada ambiente (local, staging, produção) tem o seu.

## 1. Adicionar ao .gitignore

Confirmar que o arquivo está ignorado no repositório PHP:

```
mevo-config.php
```

## 2. Criar o arquivo localmente

Na raiz do projeto PHP (mesmo diretório do `mevo-proxy.php`):

```php
<?php
// mevo-config.php — NAO versionar
return [
    // Local: aponte para a Mevo API rodando localmente ou para o ALB de staging
    'base_url' => 'http://localhost:3000',
    'api_key'  => '<valor do secret hmg/mevo-api/internal-api-key>',
];
```

### Valores por ambiente

| Ambiente | `base_url` | Fonte da `api_key` |
|---|---|---|
| Local (API local) | `http://localhost:3000` | Mesmo valor de staging |
| Local (contra staging)* | `http://internal-hmg-mevo-alb-internal-1641981751.sa-east-1.elb.amazonaws.com` | `hmg/mevo-api/internal-api-key` |
| Staging (servidor PHP) | `http://internal-hmg-mevo-alb-internal-1641981751.sa-east-1.elb.amazonaws.com` | `hmg/mevo-api/internal-api-key` |
| Produção (servidor PHP) | `http://internal-prd-mevo-alb-internal-1961577618.us-east-1.elb.amazonaws.com` | `prd/mevo-api/internal-api-key` |

\* O ALB Internal só é alcançável de dentro da VPC (`172.31.0.0/16`). Da máquina local o acesso só funciona via VPN/túnel para a VPC — na prática, para dev local rode a Mevo API localmente (`npm run dev`) e aponte para `localhost:3000`.

## 3. Obter a API Key

```bash
aws secretsmanager get-secret-value \
  --secret-id hmg/mevo-api/internal-api-key \
  --region sa-east-1 --query SecretString --output text
```

Nunca compartilhar a key por e-mail ou chat — usar canal seguro. Nos servidores, restringir permissões do arquivo:

```bash
chmod 640 mevo-config.php
```

## 4. Testar a chamada

Com a Mevo API rodando localmente:

```bash
curl -X POST http://localhost:3000/mevo/receita \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: <api_key>" \
  -d '{ ... payload ... }'
```

Respostas do middleware `verifyInternalAuth`:

| Cenário | Resposta |
|---|---|
| IP fora do CIDR permitido (SSM) | `403 Forbidden` |
| `X-Api-Key` ausente ou inválida | `401 Unauthorized` |
| Ambos válidos | Segue para o handler |

> **A verificação de CIDR é defesa em profundidade, não a barreira principal.** Atrás de qualquer ALB o IP de origem visto pelo middleware é sempre o do nó do ALB (`172.31.x.x`), então o CIDR não distingue uma chamada interna de uma externa. Quem protege as rotas internas de fato é a combinação **listener do ALB** (bloqueia o path pela internet) + **`X-Api-Key`** (credencial). Detalhes em [Autenticação](autenticacao.md).

> Em dev local, o parâmetro SSM de whitelist não existe, então a validação de IP é pulada automaticamente (`if (!process.env.INTERNAL_ALLOWED_IPS_PARAMETER) return []`). Basta a `X-Api-Key`.

## Troubleshooting

- **`403` inesperado em staging** — verifique se o IP do servidor PHP está dentro de `172.31.0.0/16` e se o parâmetro SSM `/mevo-api/staging/internal-allowed-ips` está correto. Lembre-se do cache em memória por task: mudanças no SSM podem demorar a propagar (force new deployment se necessário).
- **`401`** — a key no `mevo-config.php` diverge do secret atual. Se o secret foi rotacionado, atualize o `mevo-config.php` nos servidores e force novo deployment do ECS para renovar o cache das tasks (ver [Autenticação → Rotação de credenciais](autenticacao.md)).
- **Timeout no ALB Internal** — chamada vinda de fora da VPC, ou Security Group `*-mevo-alb-internal-sg` sem ingress TCP 80 do seu IP/CIDR de origem.
