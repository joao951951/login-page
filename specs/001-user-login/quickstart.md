# Quickstart: User Login Page

**Branch**: `001-user-login` | **Date**: 2026-04-06

## Pré-requisitos

- Node.js 22 LTS (`node --version` deve retornar `v22.x.x`)
- Docker + Docker Compose (para PostgreSQL local e Testcontainers)
- `npm` 10+

## Setup inicial

```bash
# 1. Instalar dependências
npm install

# 2. Subir PostgreSQL de desenvolvimento
docker compose up -d db

# 3. Copiar variáveis de ambiente
cp .env.example .env
# Editar .env com as credenciais locais (defaults já funcionam com docker-compose.yml)

# 4. Rodar migrações
npm run db:migrate

# 5. Iniciar servidor em modo dev
npm run dev
# → Servidor disponível em http://localhost:3000
```

## Variáveis de ambiente (`.env.example`)

```env
# Banco de dados
DATABASE_URL=postgres://login_user:login_pass@localhost:5432/login_db

# Sessão
SESSION_SECRET=mude-isso-em-producao-minimo-32-chars

# E-mail (SMTP)
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_USER=
SMTP_PASS=
EMAIL_FROM=noreply@login-page.local

# App
PORT=3000
NODE_ENV=development
```

## Docker Compose (`docker-compose.yml`)

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: login_db
      POSTGRES_USER: login_user
      POSTGRES_PASSWORD: login_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## Scripts npm

| Comando | Ação |
|---------|------|
| `npm run dev` | Servidor dev com hot-reload (nodemon) |
| `npm start` | Servidor produção |
| `npm run db:migrate` | Rodar migrações pendentes |
| `npm test` | Todos os testes (unit + integration) |
| `npm run test:unit` | Apenas testes unitários (Vitest) |
| `npm run test:integration` | Testes de integração com Testcontainers |
| `npm run test:e2e` | Testes E2E com Playwright (requer servidor rodando) |
| `npm run test:e2e:ci` | E2E com servidor iniciado automaticamente |

## Rodando os testes

```bash
# Unit + Integration (sem servidor)
npm test

# E2E (Playwright) — inicia servidor e PostgreSQL automaticamente via fixtures
npm run test:e2e:ci

# Modo interativo Playwright (útil para depurar)
npx playwright test --ui
```

**Atenção**: testes de integração usam Testcontainers — Docker deve estar rodando. Cada suite sobe um PostgreSQL isolado e destrói ao fim. Não é necessário o Docker Compose rodando para testes.

## Workflow TDD (obrigatório pela Constitution)

```
1. Escrever teste que falha (red)
   → ex: tests/integration/AuthService.test.js

2. Implementar código mínimo para passar (green)
   → ex: src/services/AuthService.js

3. Refatorar mantendo testes verdes (refactor)

4. Repeat para cada acceptance scenario do spec
```

## Estrutura de pastas relevante

```
src/
├── db/migrations/    ← adicionar SQL migrations aqui
├── routes/auth/      ← rotas de autenticação
├── services/         ← lógica de negócio (AuthService, etc.)
└── views/            ← templates HTML + htmx

tests/
├── e2e/              ← 1 arquivo por user story do spec
├── integration/      ← 1 arquivo por service
└── unit/             ← validações e utilidades puras

specs/001-user-login/
├── spec.md           ← fonte da verdade dos requisitos
├── plan.md           ← este plano
├── research.md       ← decisões técnicas documentadas
├── data-model.md     ← schema do banco
└── contracts/        ← contratos HTTP das rotas
```
