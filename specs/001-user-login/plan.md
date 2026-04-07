# Implementation Plan: User Login Page

**Branch**: `001-user-login` | **Date**: 2026-04-06 | **Spec**: [spec.md](./spec.md)  
**Input**: Feature specification from `/specs/001-user-login/spec.md`

## Summary

Implementar a página de login com autenticação por e-mail e senha, bloqueio por força bruta, sessão persistente ("Lembrar de mim") e recuperação de senha via e-mail. Stack: Node.js 22 LTS + Fastify 5 + PostgreSQL 16 + htmx 2, com testes TDD usando Vitest, Playwright e Testcontainers.

## Technical Context

**Language/Version**: Node.js 22 LTS  
**Primary Dependencies**: Fastify 5, @fastify/session, @fastify/cookie, @fastify/rate-limit, argon2, zod, htmx 2, nodemailer  
**Storage**: PostgreSQL 16 (via `postgres` driver nativo)  
**Testing**: Vitest (unit + integration), Playwright (E2E acceptance), Testcontainers Node.js (PostgreSQL real em testes de integração)  
**Target Platform**: Linux server via Docker Compose  
**Project Type**: Web application (SSR com htmx; sem SPA framework)  
**Performance Goals**: < 200 ms p95 no endpoint de login em condições normais  
**Constraints**: Rate limit de 5 tentativas / 15 min por (email + IP); token de recuperação expira em 1h e é de uso único; "Lembrar de mim" mantém sessão por 30 dias  
**Scale/Scope**: MVP — aplicação web única, autenticação email+senha somente

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Princípio | Gate | Status |
|-----------|------|--------|
| Security-First | Senha hashed com argon2 (nunca plaintext) | PASS |
| Security-First | Endpoints de auth com rate limiting (@fastify/rate-limit) | PASS |
| Security-First | Sessões com expiração configurada; token de recuperação de uso único | PASS |
| Security-First | HTTPS obrigatório; credenciais nunca em query string | PASS |
| Security-First | Validação server-side com Zod em todas as rotas de auth | PASS |
| User Experience | Mensagens de erro genéricas (nunca revelam e-mail inexistente) | PASS |
| User Experience | Formulário WCAG 2.1 AA: labels, navegação por teclado, ARIA | PASS |
| User Experience | Estados de loading com htmx (`hx-indicator`) | PASS |
| Test-Driven | TDD obrigatório: teste falha → implementa → passa → refatora | PASS |
| Test-Driven | Integração usa PostgreSQL real via Testcontainers (sem mock de auth middleware) | PASS |
| Test-Driven | Acceptance scenarios do spec mapeados 1:1 em testes Playwright | PASS |
| Simplicity | Sem SSO, OAuth, MFA — MVP email+senha apenas | PASS |
| Simplicity | Usa @fastify/session (biblioteca mantida) em vez de framework custom | PASS |
| Simplicity | Aplicação única sem microserviços | PASS |

**Resultado pré-pesquisa**: Todas as gates PASS. Nenhuma violação a justificar.

## Project Structure

### Documentation (this feature)

```text
specs/001-user-login/
├── plan.md              # Este arquivo (/speckit-plan output)
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   ├── auth.http        # Contratos das rotas HTTP
└── tasks.md             # Phase 2 output (/speckit-tasks — não gerado aqui)
```

### Source Code (repository root)

```text
src/
├── db/
│   ├── client.js           # Conexão PostgreSQL (pool)
│   └── migrations/         # SQL migrations (numeradas)
├── plugins/
│   ├── session.js          # @fastify/session + @fastify/cookie
│   ├── rate-limit.js       # @fastify/rate-limit por rota
│   └── validation.js       # Zod schema validation hook
├── routes/
│   ├── auth/
│   │   ├── login.js        # POST /auth/login, GET /auth/logout
│   │   └── password.js     # POST /auth/forgot-password, POST /auth/reset-password
│   └── index.js            # GET / (redirect se autenticado)
├── services/
│   ├── AuthService.js      # login(), logout(), isAuthenticated()
│   ├── SessionService.js   # createSession(), destroySession(), extendSession()
│   └── PasswordResetService.js  # requestReset(), validateToken(), resetPassword()
├── models/
│   ├── UserRepository.js
│   ├── LoginAttemptRepository.js
│   └── RecoveryTokenRepository.js
└── views/
    ├── login.html
    ├── forgot-password.html
    └── reset-password.html

tests/
├── e2e/                    # Playwright — acceptance scenarios (1:1 com spec)
│   ├── login.spec.js
│   ├── brute-force.spec.js
│   ├── generic-error.spec.js
│   └── password-reset.spec.js
├── integration/            # Vitest + Testcontainers (PostgreSQL real)
│   ├── AuthService.test.js
│   ├── SessionService.test.js
│   └── PasswordResetService.test.js
└── unit/                   # Vitest — lógica pura sem I/O
    ├── validation.test.js
    └── token.test.js

docker-compose.yml          # PostgreSQL dev + app
Dockerfile
package.json
```

**Structure Decision**: Aplicação web única (Option 2 adaptado para SSR). Frontend é HTML+htmx servido pelo próprio Fastify via rotas de view; não há pasta `frontend/` separada pois não há build step de SPA.

## Complexity Tracking

*Sem violações identificadas — tabela omitida conforme instrução do template.*
