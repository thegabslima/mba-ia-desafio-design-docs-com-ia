# ADR-006: Reuso máximo dos padrões existentes do projeto

- **Status:** Aceito
- **Data:** 2026-10-22
- **Decisores:** Larissa, Bruno, Diego
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-worker-separado-em-polling.md)

## Contexto

O código segue uma convenção clara: cada domínio é um módulo em `src/modules/<dominio>/` com controller, service, repository, routes e schemas (ex.: `src/modules/orders/`). Erros usam `AppError` (`src/shared/errors/app-error.ts`) e subclasses em `src/shared/errors/http-errors.ts`, tratadas por `src/middlewares/error.middleware.ts`. Logs usam Pino (`src/shared/logger/index.ts`). Autorização usa `authenticate` e `requireRole` (`src/middlewares/auth.middleware.ts`). Validação usa Zod via `src/middlewares/validate.middleware.ts`.

## Decisão

"Reuso máximo do que já existe" [09:30 Larissa]:

- Módulo `src/modules/webhooks/` com a mesma estrutura dos demais [09:27 Bruno].
- Erros do módulo estendem `AppError` com prefixo **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) [09:28 Bruno, 09:29 Larissa].
- Pino como único logger; o middleware de erro central já trata `AppError`, Zod e Prisma sem alteração [09:29 Bruno].
- Schemas Zod e `validate()` para entrada [09:30 Larissa].
- `requireRole('ADMIN')` no replay de DLQ; demais endpoints só com `authenticate` [09:36 Larissa, Sofia].
- Integração com o `OrderService` por função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação, sem injetar um repository inteiro [09:41 Bruno, Diego].
- IDs em UUID como no schema Prisma existente [09:51 Larissa].

## Alternativas Consideradas

1. **Injetar um `WebhookRepository` no `OrderService`** — descartada: uma função pura recebendo o `tx` basta [09:41 Diego].
2. **Nova biblioteca de logging ou de erros para o worker** — descartada: "não vamos botar nada novo" [09:29 Bruno].

## Consequências

**Positivas**
- Curva de aprendizado quase zero; o middleware de erro funciona sem mudanças.
- Revisão mais simples, por seguir convenções conhecidas.

**Negativas / trade-offs**
- `OrderService` passa a depender de uma função do módulo de webhooks (acoplamento unidirecional aceito).
- O `logger` redige apenas campos como `*.password` e `*.token`; a `secret` do webhook precisará entrar na lista de redação para não vazar em log.
- O worker reaproveita `env` e logger, mas precisa do seu próprio `PrismaClient`.
