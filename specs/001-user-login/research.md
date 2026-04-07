# Research: User Login Page

**Branch**: `001-user-login` | **Date**: 2026-04-06  
**Phase**: 0 — Resolve unknowns before design

---

## 1. Gerenciamento de Sessão: Server-side vs JWT

**Decision**: Server-side sessions via `@fastify/session` + PostgreSQL (connect-pg-simple como store)  
**Rationale**: JWTs não podem ser invalidados antes de expirar sem infra adicional (blocklist), o que viola o requisito de logout explícito encerrar "Lembrar de mim". Sessions server-side permitem invalidação imediata. A constitution exige que "sessões e tokens DEVEM ter expiração" e "refresh flows DEVEM ser explicitamente projetados" — mais simples com sessions do que com JWT + refresh token pairs.  
**Alternatives considered**:
- JWT stateless: Rejeitado — impossível invalidar na expiração do logout sem blocklist, que é complexidade extra sem benefício no MVP.
- `@fastify/jwt`: Rejeitado pelo mesmo motivo acima.

---

## 2. Rate Limiting: In-memory vs Persistido

**Decision**: `@fastify/rate-limit` com store em PostgreSQL via tabela `login_attempts`  
**Rationale**: Rate limiting in-memory (padrão do plugin) se perde ao reiniciar o servidor e não funciona com múltiplas instâncias. O spec exige rastrear tentativas por (email + IP) — o que precisa de persistência. A tabela `login_attempts` já será necessária para satisfazer FR-004 e FR-005; usar a mesma fonte de dados evita duplicação.  
**How**: `@fastify/rate-limit` com `keyGenerator` customizado + `onExceeding` hook; a lógica de bloqueio (5 tentativas, 15 min) fica em `LoginAttemptRepository.js` consultando PostgreSQL.  
**Alternatives considered**:
- Redis como store: Rejeitado — adiciona dependência de infraestrutura sem ganho no MVP de escala única.
- In-memory store: Rejeitado — não persiste entre reinicializações; viola SC-003 (100% das 5 tentativas devem bloquear, sem exceções).

---

## 3. "Lembrar de Mim": Implementação

**Decision**: Cookie `httpOnly; Secure; SameSite=Lax` com `maxAge` de 30 dias quando marcado; sem `maxAge` (session cookie) quando não marcado  
**Rationale**: `@fastify/session` expõe `session.options.cookie.maxAge` que pode ser definido por request. Ao marcar "Lembrar de mim", o cookie é persistente (30 dias); ao desmarcar, é um session cookie que expira ao fechar o browser. Logout explícito chama `session.destroy()` que invalida o registro no store e remove o cookie.  
**Alternatives considered**:
- Token separado no DB: Mais complexo; `@fastify/session` já resolve com store persistente.
- LocalStorage: Rejeitado — não é `httpOnly`, exposto a XSS; viola princípio Security-First.

---

## 4. Token de Recuperação de Senha

**Decision**: Token opaco de 32 bytes (`crypto.randomBytes(32).toString('hex')`), armazenado com hash SHA-256 no DB, expiração em 1h, uso único (invalidado ao usar)  
**Rationale**: Tokens opacos não vazam informação do usuário. Armazenar apenas o hash protege contra vazamento de DB (atacante com dump do DB não pode usar tokens ativos). `crypto.randomBytes(32)` garante entropia suficiente (256 bits). SHA-256 sem salt é adequado para tokens de uso único de alta entropia (diferente de senhas).  
**Alternatives considered**:
- JWT assinado com expiração: Rejeitado — válido por tempo mesmo após uso, requer blocklist para ser "de uso único" — complexidade sem benefício.
- UUID v4: Rejeitado — 122 bits de entropia é aceitável mas `randomBytes(32)` é mais idiomático para tokens de segurança em Node.js.

---

## 5. Integração com Serviço de E-mail

**Decision**: `nodemailer` com transporte SMTP configurável via variáveis de ambiente; stub/interceptor em testes  
**Rationale**: A constitution assume que "o serviço de envio de e-mails já existe ou será provido como infraestrutura". `nodemailer` é agnóstico a provedor (SMTP genérico funciona com SendGrid, SES, Mailgun, servidor próprio). Em testes de integração/E2E, usa `nodemailer`-compatible SMTP mock (ex: `smtp-server` in-memory ou Ethereal Email).  
**Alternatives considered**:
- SDK de provedor específico (SendGrid, SES): Rejeitado no MVP — acopla a um provedor específico; SMTP é universal.
- Simulação sem envio real: Adequado para testes; em produção sempre usa SMTP real configurado via env vars.

---

## 6. Acessibilidade WCAG 2.1 AA com htmx

**Decision**: HTML semântico com `<label for>`, `aria-describedby` para erros, `aria-live="polite"` para mensagens dinâmicas do htmx, `role="alert"` para erros críticos  
**Rationale**: htmx injeta HTML no DOM — leitores de tela podem não anunciar mudanças sem `aria-live`. Usando `hx-target` apontando para um `<div aria-live="polite">` garante que erros de login sejam anunciados sem quebrar o fluxo do leitor de tela. Navegação por teclado é nativa no HTML semântico (`<button type="submit">`, `<a>`, `<input>`).  
**Alternatives considered**:
- Framework JS com suporte a acessibilidade (React + Radix): Rejeitado — viola Simplicity; htmx + HTML semântico é suficiente para WCAG AA em formulários.

---

## 7. Migrações de Banco de Dados

**Decision**: Migrações SQL puras com numeração sequencial (`001_create_users.sql`, etc.), executadas via script Node.js na inicialização  
**Rationale**: Sem ORM (princípio Simplicity — sem abstração desnecessária). Script de migração simples que lê arquivos `.sql` ordenados e aplica os ainda não executados (tabela `schema_migrations` de controle). Testcontainers garante banco limpo em cada suite de testes de integração.  
**Alternatives considered**:
- Prisma/Drizzle ORM: Rejeitado — adiciona abstração e geração de código para um MVP de escala pequena; SQL direto é mais simples e mais explícito.
- Flyway/Liquibase: Rejeitado — dependência Java em projeto Node.js.

---

## Resoluções: Todos os NEEDS CLARIFICATION eliminados

| Tópico | Resolução |
|--------|-----------|
| Sessão server-side vs JWT | Server-side sessions (@fastify/session + PostgreSQL store) |
| Rate limiting store | PostgreSQL via LoginAttemptRepository |
| "Lembrar de mim" | maxAge dinâmico no cookie de sessão |
| Token de recuperação | randomBytes(32) + hash SHA-256 no DB |
| Serviço de e-mail | nodemailer SMTP + stub em testes |
| WCAG 2.1 AA com htmx | aria-live + HTML semântico |
| Migrações | SQL puro + script Node.js |
