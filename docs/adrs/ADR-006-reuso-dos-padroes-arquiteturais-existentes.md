# ADR-006: Reuso dos padrões arquiteturais existentes do OMS no módulo de webhooks

## Status

Aceito

## Contexto

Ao discutir estrutura de código, Bruno apontou que a codebase já tem um padrão claro: cada domínio é um módulo em `src/modules/<nome>` com `controller`, `service`, `repository`, `routes` e `schemas`. Ele propôs que o novo módulo de webhooks siga exatamente essa estrutura (`[09:27] Bruno`). Isso foi confirmado no código: `src/modules/orders/` contém exatamente `order.controller.ts`, `order.repository.ts`, `order.routes.ts`, `order.schemas.ts`, `order.service.ts` e `order.status.ts`, e o mesmo padrão se repete em `auth/`, `customers/`, `products/` e `users/`.

Para tratamento de erros, Bruno propôs seguir o padrão já existente de classe base `AppError` com subclasses específicas e códigos de erro em formato constante — confirmado em `src/shared/errors/app-error.ts` (classe `AppError` com `statusCode`, `errorCode`, `details`) e `src/shared/errors/http-errors.ts` (ex.: `InsufficientStockError` com código `INSUFFICIENT_STOCK`, `InvalidStatusTransitionError` com código `INVALID_STATUS_TRANSITION`). Os novos erros do módulo de webhooks devem usar o mesmo padrão, com prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) (`[09:28]-[09:29] Bruno/Larissa`).

Bruno também apontou que o logger Pino já usado no projeto inteiro (`src/shared/logger/index.ts`) não precisa de nenhuma mudança, e que o middleware de erro central (`src/middlewares/error.middleware.ts`) já trata `AppError`, `ZodError` e erros conhecidos do Prisma de forma uniforme — ele vai capturar os erros do módulo de webhooks sem precisar de alteração (`[09:29] Bruno`), o que foi confirmado no código: o middleware formata a resposta em `{ error: { code, message, details? } }` para qualquer `AppError`.

Sobre infraestrutura compartilhada, Diego perguntou se o worker abriria o mesmo `PrismaClient` da API ou um separado; Bruno respondeu que precisa ser separado, pois `PrismaClient` é por processo — mesmo banco e mesma `DATABASE_URL`, mas instância nova por ser outro processo Node (`[09:29]-[09:30] Diego/Bruno`). Larissa fechou o ponto declarando reuso máximo do que já existe: `AppError`, Pino, error middleware, padrão de módulos, padrão de schemas Zod e padrão de códigos de erro — o módulo de webhooks deve ficar estruturalmente igual aos demais módulos do projeto (`[09:30] Larissa`).

O padrão de validação de entrada com Zod também segue reaproveitado: o projeto já usa `src/middlewares/validate.middleware.ts` combinado com schemas por módulo (ex.: `src/modules/orders/order.schemas.ts`), convertendo erros de validação Zod em `ValidationError` — confirmado no código.

## Decisão

O módulo de webhooks (`src/modules/webhooks/`) segue estritamente os padrões arquiteturais já estabelecidos no projeto, sem introduzir convenções novas:

- Estrutura de módulo por domínio (`controller`, `service`, `repository`, `routes`, `schemas`), no mesmo formato de `src/modules/orders/`.
- Hierarquia de erros baseada em `AppError` (`src/shared/errors/app-error.ts`) com subclasses específicas, usando códigos de erro com prefixo `WEBHOOK_`, seguindo o padrão de `src/shared/errors/http-errors.ts`.
- Middleware de erro central (`src/middlewares/error.middleware.ts`) reaproveitado sem alteração.
- Logger Pino (`src/shared/logger/index.ts`) reaproveitado sem alteração.
- Validação de entrada via `src/middlewares/validate.middleware.ts` + schemas Zod por módulo, no mesmo padrão de `order.schemas.ts`.
- Worker com instância própria de `PrismaClient`, conectada à mesma `DATABASE_URL` da API, por ser um processo Node separado.

## Alternativas Consideradas

- **Criar convenções específicas para o módulo de webhooks** (ex.: uma hierarquia de erros própria, um logger dedicado, ou uma estrutura de pastas diferente da usada pelos demais módulos). Rejeitada implicitamente pela decisão explícita de Larissa de priorizar reuso máximo do que já existe (`[09:30] Larissa`), evitando fragmentar os padrões da base de código.

## Consequências

**Positivas:**
- Consistência com o restante da base de código, reduzindo a curva de aprendizado para qualquer engenheiro que já conheça o padrão dos módulos existentes.
- O middleware de erro central e o logger não precisam de nenhuma alteração para suportar o novo módulo, reduzindo risco de regressão em código compartilhado.
- Facilita revisão de código e onboarding, já que o módulo de webhooks é estruturalmente indistinguível dos módulos de `orders`, `customers`, `products` e `users`.

**Negativas:**
- Alguns padrões existentes (como a hierarquia de `AppError`, pensada originalmente para o ciclo request/response HTTP) precisam ser adaptados ao contexto de um worker assíncrono em background, que não responde diretamente a uma requisição HTTP.
- Constrange as escolhas de design do módulo de webhooks às convenções já fixadas pelos módulos anteriores, mesmo em pontos onde uma necessidade específica de webhooks (ex.: processamento assíncrono) poderia, em tese, justificar uma abordagem diferente.
