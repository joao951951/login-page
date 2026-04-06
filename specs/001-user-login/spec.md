# Feature Specification: User Login Page

**Feature Branch**: `001-user-login`  
**Created**: 2026-04-04  
**Status**: Draft  

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Acesso à Conta com Credenciais Válidas (Priority: P1)

Um usuário registrado acessa a página de login, informa seu e-mail e senha, e é redirecionado para sua conta com sucesso. Opcionalmente, pode marcar "Lembrar de mim" para manter a sessão ativa entre visitas.

**Why this priority**: É o fluxo principal — sem ele, a aplicação não tem valor.

**Independent Test**: Usuário informa credenciais corretas e é redirecionado para a área autenticada. Entrega acesso funcional à conta.

**Acceptance Scenarios**:

1. **Given** o usuário está na página de login, **When** informa e-mail e senha válidos e clica em "Entrar", **Then** é redirecionado para a área autenticada da aplicação.
2. **Given** o usuário marcou "Lembrar de mim" e fechou o navegador, **When** retorna ao site, **Then** permanece autenticado sem precisar fazer login novamente.
3. **Given** o usuário não marcou "Lembrar de mim", **When** fecha o navegador e retorna, **Then** é redirecionado para a página de login.

---

### User Story 2 - Proteção contra Tentativas Excessivas de Login (Priority: P2)

Após 5 tentativas consecutivas com credenciais inválidas, o sistema bloqueia temporariamente novas tentativas de login, protegendo contas contra ataques de força bruta.

**Why this priority**: Requisito de segurança explícito; protege as contas dos usuários legítimos.

**Independent Test**: Realizando 5 tentativas falhas seguidas e verificando que a 6ª é bloqueada com mensagem e tempo de espera.

**Acceptance Scenarios**:

1. **Given** o usuário tentou login 5 vezes com credenciais inválidas, **When** tenta uma 6ª vez, **Then** recebe mensagem informando que o acesso está temporariamente bloqueado e quanto tempo falta para a liberação.
2. **Given** o bloqueio está ativo, **When** o tempo de espera expira, **Then** o usuário pode tentar fazer login novamente.
3. **Given** o usuário realizou login com sucesso após tentativas falhas, **When** o sistema verifica o contador, **Then** o contador de tentativas falhas é zerado.

---

### User Story 3 - Mensagem de Erro Genérica sem Enumeração de Usuários (Priority: P2)

Quando as credenciais informadas não conferem — seja por e-mail inexistente ou senha incorreta — o sistema exibe uma única mensagem genérica, sem revelar qual dos dois campos está errado.

**Why this priority**: Requisito de segurança explícito; impede que atacantes descubram se um e-mail está ou não cadastrado.

**Independent Test**: Tentar login com e-mail inexistente e com senha errada deve produzir exatamente a mesma mensagem de erro.

**Acceptance Scenarios**:

1. **Given** o usuário informa um e-mail não cadastrado, **When** tenta fazer login, **Then** vê apenas "Credenciais inválidas" — sem indicação de que o e-mail não existe.
2. **Given** o usuário informa um e-mail válido com senha incorreta, **When** tenta fazer login, **Then** vê exatamente a mesma mensagem "Credenciais inválidas".
3. **Given** ambos os campos estão em branco, **When** o usuário tenta enviar o formulário, **Then** recebe validação indicando que os campos são obrigatórios, sem revelar dados do sistema.

---

### User Story 4 - Recuperação de Senha (Priority: P3)

Um usuário que esqueceu sua senha pode solicitar o envio de um link de redefinição para seu e-mail e criar uma nova senha.

**Why this priority**: Funcionalidade de suporte essencial, mas a aplicação já entrega valor nas histórias anteriores sem ela.

**Independent Test**: Usuário solicita recuperação, recebe link, define nova senha e consegue fazer login com ela.

**Acceptance Scenarios**:

1. **Given** o usuário está na página de login e clica em "Esqueci minha senha", **When** informa seu e-mail e confirma, **Then** recebe mensagem genérica confirmando que, se o e-mail estiver cadastrado, um link será enviado (sem revelar existência do cadastro).
2. **Given** o usuário recebeu o link de recuperação, **When** acessa o link e define uma nova senha que atende aos critérios mínimos, **Then** a senha é atualizada e ele pode fazer login com a nova senha.
3. **Given** o link de recuperação foi utilizado ou expirou, **When** o usuário tenta acessá-lo novamente, **Then** recebe mensagem informando que o link não é mais válido, com opção de solicitar um novo.

---

### Edge Cases

- O que acontece quando recuperação de senha é solicitada para e-mail não cadastrado? → Exibe a mesma mensagem de confirmação genérica, sem revelar ausência do cadastro.
- O que acontece quando o link de recuperação expira? → Link inválido com opção de solicitar um novo.
- O que acontece quando o usuário tenta fazer login durante o período de bloqueio? → Mensagem de bloqueio com tempo restante; a tentativa não conta como nova falha.
- O que acontece quando o usuário acessa a página de login já autenticado? → Redirecionamento automático para a área autenticada.
- O que acontece com o "Lembrar de mim" quando o usuário faz logout explicitamente? → A sessão persistente é encerrada; na próxima visita, o usuário precisará fazer login.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema DEVE permitir que o usuário faça login informando e-mail e senha.
- **FR-002**: O sistema DEVE exibir a mensagem "Credenciais inválidas" tanto para e-mail inexistente quanto para senha incorreta, nunca revelando qual dos dois campos está errado.
- **FR-003**: O sistema DEVE oferecer a opção "Lembrar de mim" que mantém a sessão ativa após o fechamento do navegador.
- **FR-004**: O sistema DEVE bloquear temporariamente tentativas de login após 5 falhas consecutivas, exibindo o tempo restante de bloqueio.
- **FR-005**: O sistema DEVE zerar o contador de tentativas falhas após um login bem-sucedido.
- **FR-006**: O sistema DEVE oferecer fluxo de recuperação de senha via e-mail.
- **FR-007**: A mensagem de confirmação da solicitação de recuperação de senha DEVE ser genérica, sem revelar se o e-mail está ou não cadastrado.
- **FR-008**: O link de recuperação de senha DEVE ter prazo de validade e ser de uso único.
- **FR-009**: O sistema DEVE validar que os campos e-mail e senha não estão vazios antes de submeter o formulário.
- **FR-010**: O formulário DEVE ser acessível via teclado e compatível com leitores de tela (WCAG 2.1 AA).

### Key Entities

- **Usuário**: Entidade com credenciais (e-mail e senha); possui estado de sessão e contador de tentativas falhas.
- **Sessão**: Registro de autenticação ativa; possui duração variável conforme opção "Lembrar de mim".
- **Token de Recuperação**: Código temporário de uso único vinculado a um usuário; possui prazo de validade.
- **Registro de Tentativas**: Controle de falhas de autenticação por contexto (e-mail e/ou IP); determina bloqueio temporário.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Usuários conseguem concluir o fluxo de login em menos de 30 segundos em condições normais de uso.
- **SC-002**: 100% das tentativas com credenciais inválidas retornam a mesma mensagem genérica, independentemente de o e-mail estar ou não cadastrado.
- **SC-003**: O bloqueio é ativado corretamente após exatamente 5 tentativas falhas consecutivas, sem exceções.
- **SC-004**: 100% dos links de recuperação de senha expiram após o prazo definido e tornam-se inutilizáveis após o primeiro uso.
- **SC-005**: Usuários que marcaram "Lembrar de mim" permanecem autenticados entre sessões do navegador pelo período configurado.
- **SC-006**: O formulário de login é 100% operável via teclado e não apresenta falhas de acessibilidade críticas (WCAG 2.1 AA).

## Assumptions

- O serviço de envio de e-mails já existe ou será provido como infraestrutura; não é responsabilidade desta feature implementá-lo.
- O prazo de validade do link de recuperação de senha é de 1 hora (padrão da indústria); pode ser ajustado por configuração.
- O "Lembrar de mim" mantém a sessão por 30 dias (padrão da indústria).
- O bloqueio temporário após 5 tentativas dura 15 minutos.
- O bloqueio é aplicado por combinação de e-mail e endereço de origem, para evitar bloqueio indevido causado por atacantes que conhecem o e-mail da vítima.
- A página de login é a única forma de autenticação no MVP; SSO e login social estão fora do escopo.
- Usuário já autenticado que acessa a página de login é redirecionado automaticamente para a área logada.
