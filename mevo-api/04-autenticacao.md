# Autenticação

A Mevo API tem **duas rotas de entrada independentes**, cada uma com seu middleware. A proteção das rotas internas contra acesso externo é feita em **duas camadas**: separação de rotas no listener do ALB (rede) e o middleware `verifyInternalAuth` (aplicação).

## Visão geral

| | Rota externa (webhook) | Rota interna (endpoints) |
|---|---|---|
| Chamador | Mevo (empresa) | Backend PHP do Webdental |
| Entrada | Cloudflare Proxy → ALB External | `mevo-proxy.php` → ALB Internal |
| Middleware | `verifyMevoWebhook` | `verifyInternalAuth` |
| Verificação de rede | `CF-Connecting-IP` na whitelist de IPs da Mevo (SSM) | IP de origem dentro do CIDR `172.31.0.0/16` (SSM) |
| Verificação de credencial | `Authorization: Bearer {token}` — secret `*/mevo-api/webhook` | Header `X-Api-Key` — secret `*/mevo-api/internal-api-key` |
| Falha de rede | `403 Forbidden` | `403 Forbidden` |
| Falha de credencial | `401 Unauthorized` | `401 Unauthorized` |
| Rota protegida | `POST /webhook/*` | `/mevo/*`, `/paciente/*`, `/documento/*`, `/receita/*` |

## Separação de rotas no listener (ALB External)

Esta é a **primeira camada** de proteção das rotas internas contra a internet, e a mais importante.

O ALB External e o ALB Internal encaminham para o **mesmo target group** — o container Express atende todas as rotas em ambos. Sem separação, qualquer rota interna (`/mevo/*`, `/paciente/*`, `/documento/*`, `/receita/*`) seria alcançável pela internet através do ALB External. O middleware sozinho **não** basta para bloqueá-las: o IP de origem visto pelo container é sempre o do nó do ALB (`172.31.x.x`) nos dois ALBs, então a verificação de CIDR não distingue interno de externo.

A separação é feita no listener **HTTPS:443** do ALB External:

| Prioridade | Condição | Ação |
|---|---|---|
| 10 | Path `/webhook/*` | Forward → target group |
| default | (todo o resto) | Fixed response **403** |

Com isso, apenas o webhook é público. Todas as rotas internas caem no 403 do ALB **antes de chegar no container**. O padrão é uma **allowlist** (libera só `/webhook/*`) e não uma denylist: rotas internas novas nascem protegidas automaticamente, sem precisar atualizar a regra.

> **Dependência crítica:** a decisão de manter o Security Group do ALB aberto (`0.0.0.0/0`) é segura **porque** este listener existe. Alterar a regra `/webhook/*` ou o default action reabre as rotas internas para a internet. Não modificar o listener sem revisar esta dependência.

Distinção útil no diagnóstico: o 403 do listener **não** traz o header `X-Powered-By: Express` (é o ALB respondendo). Um 403 com `X-Powered-By` veio do middleware, dentro do container.

## Rota externa — `verifyMevoWebhook`

1. O Cloudflare Proxy está ativo nos **dois** ambientes, então o IP que chega no ALB é sempre do Cloudflare. O IP **real** da Mevo vem no header `CF-Connecting-IP` e é comparado com a whitelist armazenada no SSM.
2. O token do webhook é validado no header `Authorization: Bearer {token}` contra o secret do Secrets Manager.

> ⚠ **A autorização por IP usa exclusivamente `CF-Connecting-IP`, nunca `X-Forwarded-For`.** O ALB *acrescenta* o IP da conexão ao final do XFF em vez de sobrescrevê-lo, então o primeiro elemento da lista é controlado pelo cliente — um chamador define `X-Forwarded-For: <IP permitido>` e passaria pela whitelist. Só o `CF-Connecting-IP` é sobrescrito pelo Cloudflare e é confiável. O `getClientIp` compartilhado (`helpers/get-client-ip.ts`) lê `CF-Connecting-IP` com fallback para o IP da conexão TCP; o XFF não é lido. Comportamento idêntico nos dois ambientes.

## Rota interna — `verifyInternalAuth`

1. **Whitelist de rede por CIDR**: o IP de origem precisa estar dentro do CIDR configurado no SSM Parameter Store:
   - Staging: `/mevo-api/staging/internal-allowed-ips` → `172.31.0.0/16`
   - Produção: `/mevo-api/production/internal-allowed-ips` → `172.31.0.0/16`
2. **API Key**: o header `X-Api-Key` é validado contra o secret:
   - Staging: `hmg/mevo-api/internal-api-key`
   - Produção: `prd/mevo-api/internal-api-key`

> A verificação de CIDR é **defesa em profundidade**, não a barreira principal. Atrás do ALB (interno ou externo), o IP de origem é sempre o do nó do ALB (`172.31.x.x`), então o CIDR não separa chamada interna de externa. Quem protege as rotas internas de fato é a combinação **listener** (bloqueia o path pela internet) + **`X-Api-Key`** (credencial). O CIDR sozinho passaria para qualquer requisição vinda de um ALB.

O `getClientIp` compartilhado também é usado aqui — lê `CF-Connecting-IP` com fallback para o socket, sem ler XFF.

## Limitação conhecida — cache por task

Os middlewares fazem cache em memória dos valores de SSM e Secrets Manager. Com múltiplas tasks (Auto Scaling em produção: min 2, max 5), uma rotação de API Key ou mudança de CIDR propaga de forma **inconsistente** entre as tasks até o TTL do cache (5 min) expirar em cada uma.

**Mitigação planejada:** migrar o cache para Valkey/ElastiCache compartilhado (`app-valkey-webdental-prod`) — pendência registrada.

## Rotação de credenciais

1. Gerar novo valor e atualizar o secret no Secrets Manager
2. Atualizar o `mevo-config.php` nos servidores PHP (rota interna) ou comunicar a Mevo (rota externa)
3. Forçar novo deployment do ECS Service para renovar o cache de todas as tasks:
   ```bash
   aws ecs update-service --cluster prd-mevo-ecs-cluster \
     --service prd-mevo-api-service --force-new-deployment
   ```
