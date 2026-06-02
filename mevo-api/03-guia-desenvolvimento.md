# Guia de Desenvolvimento — Mevo API

## Pré-requisitos

| Ferramenta | Versão mínima | Instalação |
|---|---|---|
| Node.js | 20.x | https://nodejs.org |
| Docker Desktop | 24.x | https://docker.com |
| AWS CLI | 2.x | https://aws.amazon.com/cli |
| Git | 2.x | https://git-scm.com |

---

## Setup do Ambiente Local

### 1. Clonar o repositório

```bash
git clone https://github.com/gerfinanceirosolucoes/mevo_api.git
cd mevo_api
```

### 2. Instalar dependências

```bash
npm install
```

### 3. Configurar variáveis de ambiente

Crie o arquivo `.env` na raiz do projeto (nunca commitar):

```env
# Aplicação
NODE_ENV=development
APP_PORT=3000
APP_ORIGIN=*

# Banco de dados
DB_HOST=IP_DO_BANCO_STAGING
DB_PORT=3306
DB_USERNAME=mevo_api
DB_PASSWORD=SUA_SENHA
DB_DATABASE=NOME_DO_BANCO

# AWS — credenciais do stg-mevo-s3-user (solicitar ao TechLead)
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=sa-east-1

# S3
AWS_BUCKET=stg-webdental-mevo-docs-843630347602-sa-east-1-an
S3_PREFIX=dev-SEU_NOME

# Mevo
MEVO_API_URL=https://api.mevo.com.br
MEVO_USERNAME=...
MEVO_PASSWORD=...
MEVO_WEBHOOK_TOKEN=...
```

> **Atenção:** solicite as credenciais ao TechLead via canal seguro. Nunca compartilhe por e-mail ou chat.

### 4. Rodar em modo desenvolvimento

```bash
npm run dev
```

O servidor inicia em `http://localhost:3000`. Alterações no código recarregam automaticamente via `tsx watch`.

---

## Scripts Disponíveis

| Script | Comando | O que faz |
|---|---|---|
| Desenvolvimento | `npm run dev` | Inicia com hot-reload via tsx |
| Build | `npm run build` | Compila TypeScript para `dist/` |
| Validação completa | `npm run validate` | Build + Docker build + Docker run |
| Formatação | `npm run format` | Formata o código com Prettier |

---

## npm run validate

O `validate` é a **fonte da verdade** antes de qualquer push para staging. Ele simula exatamente o que acontece em produção:

```bash
npm run validate
```

Resultado esperado:
```
> tsc                          ← TypeScript sem erros
[+] Building ... FINISHED      ← Imagem Docker construída
Database connection failed     ← Esperado (sem banco local)
API rodando 3000               ← Container subiu corretamente
```

> Se o `validate` passar, o código está pronto para o merge em staging.

---

## Estrutura do Projeto

```
src/
  clients/        ← comunicação com APIs externas (Mevo)
  config/         ← variáveis de ambiente (env.ts)
  constants/      ← constantes da aplicação
  controllers/    ← entrada HTTP, sem lógica de negócio
  database/       ← configuração do TypeORM (data-source.ts)
  entities/       ← entidades do banco de dados
  helpers/        ← funções utilitárias
  mappers/        ← transformação de dados (Mevo → domínio interno)
  middleware/     ← validações transversais HTTP
  providers/      ← provedores externos (S3, storage local)
  repositories/   ← acesso ao banco de dados
  routes/         ← definição das rotas (routes.ts)
  schemas/        ← schemas de validação Zod
  services/       ← regras de negócio
  index.ts        ← entry point da aplicação
```

---

## Padrões de Código

### Conventional Commits

Todos os commits devem seguir o padrão Conventional Commits. O `commitlint` está configurado e bloqueia commits fora do padrão.

| Tipo | Quando usar |
|---|---|
| `feat` | Nova funcionalidade |
| `fix` | Correção de bug |
| `chore` | Manutenção, configuração, dependências |
| `refactor` | Refatoração sem nova funcionalidade |
| `docs` | Documentação |
| `build` | Mudanças no processo de build |
| `test` | Adição ou correção de testes |

**Exemplos:**
```bash
git commit -m "feat: adiciona endpoint de cancelamento de receita"
git commit -m "fix: corrige tipagem no finalizar-prescricao-service"
git commit -m "chore: atualiza dependências do projeto"
```

### TypeScript Strict

O projeto usa TypeScript com configuração estrita. Algumas regras importantes:

```typescript
// ❌ Não usar — pode causar erro em runtime
const nome = usuario.nome;

// ✅ Usar — trata o caso undefined
const nome = usuario.nome ?? '';
const nome = usuario.nome!; // apenas se tiver certeza que existe

// ❌ Não usar com verbatimModuleSyntax
import { Request } from 'express';

// ✅ Usar para tipos apenas
import type { Request } from 'express';

// ❌ Dependência circular — causa erro em runtime com node puro
import { Prescricao } from './prescricao.js';
@ManyToOne(() => Prescricao)

// ✅ Resolver dependência circular
import type { Prescricao } from './prescricao.js';
@ManyToOne('Prescricao')
```

### Imports com .js

O projeto usa ES Modules com `"type": "module"`. Todos os imports de arquivos locais precisam da extensão `.js` (mesmo sendo `.ts`):

```typescript
// ✅ Correto
import { ReceitaController } from '../controllers/receita-controller.js';

// ❌ Errado — vai falhar em runtime
import { ReceitaController } from '../controllers/receita-controller';
```

---

## Fluxo de Branches

```
main           ← código de produção
  │
  └── staging  ← ambiente de homologação (deploy automático)
        │
        └── feat/minha-feature  ← desenvolvimento
```

**Regras:**
- Nunca commitar diretamente em `staging` ou `main`
- Sempre criar uma branch `feat/*` ou `fix/*` a partir de `staging`
- Rodar `npm run validate` antes do merge
- Abrir PR para `staging` — solicitar revisão do TechLead

---

## Credenciais AWS Locais

O projeto usa o AWS SDK v3 que resolve credenciais automaticamente nesta ordem:

1. Variáveis de ambiente (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
2. `~/.aws/credentials`
3. IAM Role (ECS Fargate em staging/produção)

Em desenvolvimento local, coloque as credenciais do `stg-mevo-s3-user` no `.env`. Em staging e produção, o container usa a IAM Role automaticamente — sem credenciais no código.
