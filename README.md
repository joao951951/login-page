# login-page

Projeto de exemplo para testar o [SpecKit](https://github.com/github-spec-kit/spec-kit) — ferramenta de workflow de especificação e planejamento para desenvolvimento orientado a specs.

## O que é este projeto?

Uma página de login simples (email + senha) usada como caso de teste real do SpecKit. O objetivo não é a aplicação em si, mas validar o fluxo completo da ferramenta:

```
/speckit-specify → /speckit-plan → /speckit-tasks → /speckit-implement
```

## Stack

- **Backend**: Node.js 22 LTS + Fastify 5
- **Frontend**: HTML + htmx 2 (SSR, sem SPA)
- **Banco de dados**: PostgreSQL 16
- **Testes**: Vitest (unit/integration) + Playwright (E2E) + Testcontainers

## Funcionalidades especificadas

- Login com e-mail e senha
- Bloqueio após 5 tentativas falhas (15 min)
- Mensagem de erro genérica (sem enumerar usuários)
- "Lembrar de mim" (sessão de 30 dias)
- Recuperação de senha via e-mail (token de uso único, expira em 1h)

## Estrutura de specs

```
specs/001-user-login/
├── spec.md         # Requisitos e acceptance scenarios
├── plan.md         # Plano técnico (stack, estrutura, constitution check)
├── research.md     # Decisões técnicas documentadas
├── data-model.md   # Entidades e schema SQL
├── quickstart.md   # Como rodar o projeto
├── contracts/      # Contratos HTTP das rotas
└── tasks.md        # Tarefas de implementação (TDD)
```

## SpecKit workflow executado

| Comando | Status |
|---------|--------|
| `/speckit-specify` | Concluido |
| `/speckit-plan` | Concluido |
| `/speckit-tasks` | Pendente |
| `/speckit-implement` | Pendente |
