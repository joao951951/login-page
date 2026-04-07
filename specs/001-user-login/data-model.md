# Data Model: User Login Page

**Branch**: `001-user-login` | **Date**: 2026-04-06  
**Phase**: 1 — Design & Contracts

---

## Entities

### 1. User

Representa um usuário registrado com credenciais de acesso.

| Campo | Tipo | Constraints | Notas |
|-------|------|-------------|-------|
| `id` | `UUID` | PK, NOT NULL, DEFAULT gen_random_uuid() | Identificador opaco |
| `email` | `VARCHAR(254)` | UNIQUE, NOT NULL | Normalizado em lowercase antes de salvar |
| `password_hash` | `VARCHAR(255)` | NOT NULL | argon2id hash; nunca plaintext |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | Atualizado via trigger |

**Validation rules**:
- `email` deve ser válido conforme RFC 5321 (validado com Zod `z.string().email()`)
- `password` na requisição deve ter no mínimo 8 caracteres (validado antes do hash)
- `email` normalizado para lowercase no service antes de qualquer operação

**State transitions**: Nenhuma máquina de estados — entidade estática no MVP.

---

### 2. Session

Representa uma sessão de autenticação ativa. Gerenciada pelo `@fastify/session` com store PostgreSQL (`connect-pg-simple`).

| Campo | Tipo | Constraints | Notas |
|-------|------|-------------|-------|
| `sid` | `VARCHAR(255)` | PK, NOT NULL | ID da sessão (gerado pelo @fastify/session) |
| `sess` | `JSONB` | NOT NULL | Dados da sessão serializado (inclui `userId`, `rememberMe`) |
| `expire` | `TIMESTAMPTZ` | NOT NULL | Timestamp de expiração |

**Notas**:
- Tabela gerenciada automaticamente pelo `connect-pg-simple`; não manipulada diretamente pelo código da aplicação.
- `expire` = now() + 30 dias (se "Lembrar de mim") ou now() + duração da sessão de browser (sem maxAge fixo).
- Invalidação imediata via `session.destroy()` no logout.

---

### 3. LoginAttempt

Rastreia tentativas de login falhas para o mecanismo de rate limiting (FR-004, FR-005).

| Campo | Tipo | Constraints | Notas |
|-------|------|-------------|-------|
| `id` | `UUID` | PK, NOT NULL, DEFAULT gen_random_uuid() | |
| `email` | `VARCHAR(254)` | NOT NULL | E-mail informado na tentativa (normalizado) |
| `ip_address` | `INET` | NOT NULL | IP de origem da requisição |
| `attempted_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `succeeded` | `BOOLEAN` | NOT NULL, DEFAULT false | `true` apenas no login bem-sucedido |

**Validation rules**:
- Bloqueio ativado quando `COUNT(*) >= 5` WHERE `email = $1 AND ip_address = $2 AND succeeded = false AND attempted_at > now() - interval '15 minutes'`
- Contador "zerado" via login bem-sucedido: insere registro com `succeeded = true`; as contagens ignoram registros anteriores ao último `succeeded = true`

**State transitions**:
```
[tentativa falha] → registra succeeded=false
[5 tentativas falhas em 15min] → bloqueio ativo (consulta, não estado persistido)
[login bem-sucedido] → registra succeeded=true (zera janela de contagem)
[15min expiram] → desbloqueio automático (janela de tempo desliza)
```

---

### 4. RecoveryToken

Token de uso único para redefinição de senha (FR-006, FR-007, FR-008).

| Campo | Tipo | Constraints | Notas |
|-------|------|-------------|-------|
| `id` | `UUID` | PK, NOT NULL, DEFAULT gen_random_uuid() | |
| `user_id` | `UUID` | FK → users.id, NOT NULL | |
| `token_hash` | `VARCHAR(64)` | NOT NULL | SHA-256 hex do token opaco de 32 bytes |
| `expires_at` | `TIMESTAMPTZ` | NOT NULL | now() + 1 hora |
| `used_at` | `TIMESTAMPTZ` | NULL | Preenchido ao usar; NULL = não usado |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Validation rules**:
- Token válido: `used_at IS NULL AND expires_at > now()`
- Ao usar: `UPDATE recovery_tokens SET used_at = now() WHERE id = $1`
- Múltiplos tokens ativos por usuário são permitidos (solicitações repetidas); apenas o usado primeiro é consumido

**State transitions**:
```
[gerado] → used_at=NULL, expires_at=now()+1h (VÁLIDO)
[usado]  → used_at=now() (INVÁLIDO — uso único)
[expirado] → expires_at < now() (INVÁLIDO — tempo)
```

---

## Relationships

```
users (1) ──────< (N) recovery_tokens
users (1) ──────< (N) login_attempts   [via email — não FK direta, pois email pode não existir]
users (1) ──────< (N) sessions         [via sess.userId armazenado no JSONB]
```

**Nota sobre login_attempts**: Não há FK para `users` porque tentativas de login com e-mail inexistente também devem ser registradas (para rate limiting correto e para não revelar se o e-mail existe).

---

## SQL Schema

```sql
-- migrations/001_create_users.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         VARCHAR(254) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);

-- migrations/002_create_sessions.sql
-- (gerenciada pelo connect-pg-simple — schema gerado automaticamente)
-- Referência: https://github.com/voxpelli/node-connect-pg-simple

-- migrations/003_create_login_attempts.sql
CREATE TABLE login_attempts (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email        VARCHAR(254) NOT NULL,
  ip_address   INET NOT NULL,
  attempted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  succeeded    BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_login_attempts_lookup
  ON login_attempts (email, ip_address, attempted_at, succeeded);

-- migrations/004_create_recovery_tokens.sql
CREATE TABLE recovery_tokens (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash VARCHAR(64) NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  used_at    TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recovery_tokens_hash ON recovery_tokens (token_hash);
CREATE INDEX idx_recovery_tokens_user  ON recovery_tokens (user_id);
```
