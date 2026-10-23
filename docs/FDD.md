# FDD — Sistema de Webhooks de Notificação de Pedidos

Feature Design Document. Responde "como construir". Decisões estão nos [ADRs](adrs/); a proposta e as questões em aberto estão no [RFC](RFC.md). Itens marcados **[design]** são detalhes de implementação deduzidos pelo autor, sem decisão explícita na reunião, e devem ser validados na revisão.

## 1. Contexto e motivação técnica

`OrderService.changeStatus` (`src/modules/orders/order.service.ts`) muda o status dentro de `prisma.$transaction` e não emite nenhum sinal externo. Clientes B2B fazem polling em `GET /orders` [09:00 Marcos]. Precisamos emitir um evento por mudança de status, de forma atômica com a transação e entregue de forma assíncrona em menos de 10 s [09:02 Marcos].

## 2. Objetivos técnicos

- Nenhuma mudança de status sem evento registrado (e vice-versa) — atomicidade [09:40 Bruno].
- Latência de entrega < 10 s no caminho feliz (polling de 2 s) [09:09 Diego].
- At-least-once com `X-Event-Id` [09:25 Diego]; ordenação por `order_id` com um worker [09:12 Diego].
- Reuso dos padrões do projeto [09:30 Larissa].

## 3. Escopo e exclusões

**Dentro:** tabelas, módulo `webhooks`, worker, CRUD de webhook, rotação de secret, histórico de entregas, replay de DLQ.

**Fora:** e-mail de alerta [09:37 Larissa]; rate limiting de saída [09:39 Larissa]; dashboard visual [09:40 Larissa]; arquivamento de linhas entregues [09:08 Diego]; múltiplos workers [09:13 Diego]; recebimento de webhooks de clientes [09:02 Marcos].

## 4. Modelo de dados (`prisma/schema.prisma`)

Todos os ids UUID `@db.Char(36)` [09:51 Larissa]. Nomes de tabelas citados na reunião: `webhook_outbox`, `webhook_dead_letter`; as demais são **[design]**.

| Tabela | Campos principais |
|---|---|
| `webhook_endpoints` **[design]** | `id`, `customer_id` (FK `customers`), `url`, `secret`, `previous_secret?`, `previous_secret_expires_at?`, `events` (JSON: lista de `OrderStatus`), `active`, `created_at`, `updated_at` [09:21 Bruno/Sofia, 09:33 Marcos] |
| `webhook_outbox` | `id`, `event_id` (UUID), `webhook_id` (FK), `order_id`, `payload` (JSON snapshot), `status` (`PENDING`/`PROCESSING`/`FAILED`/`DELIVERED`), `attempts`, `next_attempt_at`, `last_error?`, `created_at`, `updated_at`. Índices: `status`, `created_at` [09:08 Diego]; `(status, next_attempt_at)` **[design]** |
| `webhook_dead_letter` | `id`, `outbox_id`, `webhook_id`, `payload`, `failure_reason`, `failed_at` [09:18 Diego] |
| `webhook_deliveries` **[design]** | `id`, `webhook_id`, `event_id`, `attempt`, `success`, `status_code?`, `response_body?` (truncado), `duration_ms`, `payload`, `created_at` — sustenta o histórico "sucesso/falha, payload, response, tempo de resposta" [09:34 Marcos] |

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (síncrono, dentro da transação)

1. `OrderService.changeStatus` valida a transição e executa estoque/`orders`/`order_status_history` como hoje.
2. Antes de retornar de dentro do `$transaction`, chama `publishWebhookEvent(tx, order, from, to)` [09:41 Bruno].
3. A função busca `webhook_endpoints` com `customer_id = order.customerId`, `active = true` e `to` presente em `events` [09:33–09:34].
4. Para cada endpoint, monta o payload (seção 6.2), gera `event_id = uuid`, valida tamanho ≤ 64 KB (senão lança `WebhookPayloadTooLargeError`, causando rollback) e insere em `webhook_outbox` com `status = PENDING`, `attempts = 0`, `next_attempt_at = now()`.
5. Nenhum endpoint interessado → nenhuma linha [09:34 Bruno].
6. Qualquer erro propaga e desfaz a transação inteira [09:40 Bruno].

### 5.2 Processamento pelo worker

Loop em `src/worker.ts`, intervalo de 2 s [09:09 Diego]:

1. Seleciona um batch pequeno (default 10 **[design]**) com `status = PENDING AND next_attempt_at <= now()` ordenado por `created_at` [09:12 Diego]. Marca como `PROCESSING` na mesma transação curta.
2. Para cada linha: carrega o endpoint, serializa o payload, calcula `X-Signature` e faz `POST` com timeout de 10 s [09:42 Diego].
3. Resposta 2xx → `DELIVERED`, registra em `webhook_deliveries`.
4. Qualquer outra coisa (não‑2xx, timeout, erro de rede) → registra falha em `webhook_deliveries` e segue para 5.3.
5. Ao iniciar, linhas presas em `PROCESSING` (crash do worker) voltam a `PENDING` **[design]**; isso é coberto pelo at-least-once.
6. SIGINT/SIGTERM: termina o ciclo corrente e chama `prisma.$disconnect()`, como em `src/server.ts`.

### 5.3 Retry

- `attempts` é incrementado a cada falha. Atrasos: `[1m, 5m, 30m, 2h, 12h]` [09:17 Larissa]. `next_attempt_at = now + atraso[attempts-1]`, `status = PENDING`.
- Interpretação adotada (RFC Q-01): 1 envio inicial + 5 retries; a 6ª falha esgota. Se a interpretação mudar, só a constante `RETRY_DELAYS_MS` muda.

### 5.4 DLQ e replay

- Esgotado o teto: dentro de uma transação, copia a linha para `webhook_dead_letter` (payload, motivo, timestamp) e marca a outbox `FAILED` [09:18 Diego].
- `POST /admin/webhooks/dead-letter/:id/replay` (ADMIN): cria nova linha `PENDING` na outbox (ou reaproveita a original com `attempts = 0`) **[design]**, mantendo o mesmo `event_id`, e loga `userId` do operador [09:36 Sofia].

### 5.5 Rotação de secret

`POST /webhooks/:id/rotate-secret` **[design: rota]** gera nova secret; a atual vira `previous_secret` com `previous_secret_expires_at = now + 24h` [09:21 Sofia]. A secret só é exibida na criação e na rotação.

## 6. Contratos públicos

Prefixo real da aplicação: `/api/v1` (`src/app.ts`). Rotas abaixo relativas a ele. Todas exigem `Authorization: Bearer <JWT>` (`authenticate`); erros no formato do `error.middleware.ts`: `{ "error": { "code", "message", "details?" } }`.

### 6.1 Endpoints

**1. `POST /webhooks`** — cria. `customerId` vem no body, não do JWT [09:32 Larissa].

```json
// request
{ "customerId": "0b6c…", "url": "https://api.atlas.com.br/hooks/oms", "events": ["SHIPPED", "DELIVERED"] }
// 201
{ "id": "7f1e…", "customerId": "0b6c…", "url": "https://api.atlas.com.br/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"], "active": true, "secret": "whsec_…", "createdAt": "2026-10-22T12:00:00.000Z" }
```
Status: 201; 400 `VALIDATION_ERROR` / `WEBHOOK_INVALID_URL` (não https); 404 `NOT_FOUND` (customer); 401.

**2. `GET /webhooks?customerId=<uuid>`** — lista os webhooks do customer [09:33 Bruno]. A `secret` não é retornada.

```json
// 200
{ "data": [ { "id": "7f1e…", "customerId": "0b6c…", "url": "https://…", "events": ["SHIPPED"], "active": true } ],
  "page": 1, "pageSize": 20, "total": 1 }
```
(formato de paginação de `src/shared/http/response.ts`). Status: 200; 400; 401.

**3. `PATCH /webhooks/:id`** — edita `url`, `events`, `active` [09:33 Bruno].

```json
// request
{ "events": ["PAID", "SHIPPED"], "active": true }
// 200
{ "id": "7f1e…", "url": "https://…", "events": ["PAID", "SHIPPED"], "active": true }
```
Status: 200; 400 `WEBHOOK_INVALID_URL`; 404 `WEBHOOK_NOT_FOUND`.

**4. `DELETE /webhooks/:id`** — remove [09:33 Bruno]. Request sem body. Response: 204 sem corpo. Status: 204; 404 `WEBHOOK_NOT_FOUND`. Eventos já na outbox deste endpoint deixam de ser enviados **[design]**.

**5. `POST /webhooks/:id/rotate-secret`** **[design: rota]** — rotaciona [09:21 Sofia].

```json
// request: {}
// 200
{ "id": "7f1e…", "secret": "whsec_novo…", "previousSecretExpiresAt": "2026-10-23T12:00:00.000Z" }
```
Status: 200; 404 `WEBHOOK_NOT_FOUND`.

**6. `GET /webhooks/:id/deliveries`** — últimas 100 entregas [09:34 Marcos].

```json
// 200
{ "data": [ { "eventId": "c2d4…", "attempt": 1, "success": false, "statusCode": 503,
    "durationMs": 412, "responseBody": "Service Unavailable",
    "payload": { "event_type": "order.status_changed" }, "createdAt": "2026-10-22T12:01:00.000Z" } ] }
```
Status: 200; 404 `WEBHOOK_NOT_FOUND`.

**7. `POST /admin/webhooks/dead-letter/:id/replay`** — somente `ADMIN` (`requireRole('ADMIN')`) [09:36 Larissa].

```json
// request: sem body
// 202
{ "deadLetterId": "9a8b…", "eventId": "c2d4…", "status": "PENDING" }
```
Status: 202; 401; 403 `FORBIDDEN`; 404 `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 6.2 Contrato de saída (chamada ao cliente)

Headers [09:44 Diego, Sofia]:

```
POST <url do cliente>
Content-Type: application/json
X-Event-Id: <uuid>
X-Signature: <hex hmac-sha256 do corpo>
X-Timestamp: <ISO 8601 do envio>
X-Webhook-Id: <id do endpoint>
```

Corpo [09:43 Diego]:

```json
{ "event_id": "c2d4…", "event_type": "order.status_changed", "timestamp": "2026-10-22T12:00:00.000Z",
  "order_id": "…", "order_number": "ORD-000123", "from_status": "PROCESSING", "to_status": "SHIPPED",
  "customer_id": "0b6c…", "total_cents": 125000 }
```
Sem `items`. Semântica: 2xx = sucesso; qualquer outro resultado = falha com retry. Formato da assinatura (hex) e prefixo `whsec_` são **[design]**. Durante o grace period, proposta: `X-Signature` com as duas assinaturas separadas por vírgula (RFC Q-03).

## 7. Matriz de erros

Classes em `src/modules/webhooks/webhook.errors.ts` (novo), estendendo `AppError` / `NotFoundError` / `UnprocessableEntityError` etc. (`src/shared/errors/http-errors.ts`). Os três primeiros códigos foram citados na reunião [09:28 Bruno]; o restante é **[design]** no mesmo padrão.

| Código | HTTP | Quando |
|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook inexistente |
| `WEBHOOK_INVALID_URL` | 400 | URL não https ou inválida [09:23 Sofia] |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação que exige secret sem secret disponível |
| `WEBHOOK_INVALID_EVENTS` | 400 | `events` vazio ou com status inválido |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload > 64 KB [09:24 Larissa] |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de id inexistente |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno) | Cliente não respondeu em 10 s [09:42 Diego] |
| `WEBHOOK_DELIVERY_FAILED` | — (interno) | Resposta não‑2xx ou erro de rede |
| `WEBHOOK_RETRY_EXHAUSTED` | — (interno) | Tentativas esgotadas → DLQ |

Códigos "internos" aparecem em `last_error`, `failure_reason` e logs, não em respostas HTTP.

## 8. Estratégias de resiliência

| Aspecto | Estratégia |
|---|---|
| Timeout | 10 s por chamada [09:42 Diego] |
| Retry/backoff | 1m, 5m, 30m, 2h, 12h; teto de 5 retries [09:17 Larissa] |
| Fallback | DLQ + replay manual ADMIN [09:18 Diego] |
| Crash do worker | Recuperação de `PROCESSING` presos no start; duplicata coberta pelo at-least-once |
| Falha do banco | Worker loga e tenta no próximo ciclo; API não é afetada |
| Backlog | Batch pequeno por ciclo; sem rate limiting nesta fase [09:39 Larissa] |
| Payload grande | Erro, sem truncar [09:23 Sofia] |

## 9. Observabilidade

- **Logs (Pino, `src/shared/logger/index.ts`):** eventos `webhook_event_enqueued`, `webhook_delivery_succeeded`, `webhook_delivery_failed`, `webhook_moved_to_dlq`, `webhook_replayed` (com `userId`, exigência de auditoria [09:36 Sofia]). Campos: `eventId`, `webhookId`, `orderId`, `attempt`, `statusCode`, `durationMs`. Adicionar `*.secret`, `*.previousSecret` e `X-Signature` à lista `redactPaths` do logger **[design]**.
- **Métricas [design]:** `webhook_outbox_pending` (gauge), `webhook_oldest_pending_age_seconds` (gauge), `webhook_deliveries_total{result}`, `webhook_delivery_duration_ms` (histograma), `webhook_dead_letter_total`. O projeto não tem biblioteca de métricas; até existir, derivar de queries sobre `webhook_outbox`/`webhook_deliveries` ou logs.
- **Tracing [design]:** o projeto não tem OpenTelemetry. Correlação via `event_id` (propagado em log e header `X-Event-Id`) e `req.id` do `request-logger.middleware.ts` gravado no payload de log do enfileiramento. Instrumentação distribuída fica como evolução.
- **Alerta sugerido:** idade do evento pendente mais antigo > 10 s sustentado (requisito de tempo real).

## 10. Integração com o sistema existente

| Arquivo | Como o módulo se integra |
|---|---|
| `src/modules/orders/order.service.ts` | Em `changeStatus`, chamar `publishWebhookEvent(tx, order, from, to)` dentro do `$transaction`, após `orderStatusHistory.create` e antes do `refreshed`. Usa o `TxClient` já definido; sem injetar repository no construtor. |
| `prisma/schema.prisma` | Novos models `WebhookEndpoint`, `WebhookOutbox`, `WebhookDeadLetter`, `WebhookDelivery` (`@id @default(uuid()) @db.Char(36)`, `@@map` snake_case) e relação com `Customer`; nova migration em `prisma/migrations/`. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Erros `Webhook*Error` estendem `AppError`/`NotFoundError`/`UnprocessableEntityError`; exportados de `src/shared/errors/index.ts` ou do próprio módulo. `src/middlewares/error.middleware.ts` já os serializa, sem mudança. |
| `src/middlewares/auth.middleware.ts` | Todas as rotas usam `authenticate`; replay usa `requireRole('ADMIN')`. |
| `src/middlewares/validate.middleware.ts` | `validate({ body, params, query })` com schemas Zod em `webhook.schemas.ts` (URL `https`, `events` com `z.nativeEnum(OrderStatus)`). |
| `src/routes/index.ts` e `src/app.ts` | Registrar `buildWebhookRouter` em `/webhooks` e `/admin/webhooks`; instanciar repository/service/controller em `buildControllers`; incluir `webhooks` no tipo `Controllers`. |
| `src/server.ts` | Modelo para o novo `src/worker.ts`: bootstrap, `logger`, shutdown em SIGINT/SIGTERM, `$disconnect()`. |
| `src/config/database.ts` e `src/config/env.ts` | O worker chama `createPrismaClient()` para ter instância própria [09:30 Bruno]. Novas variáveis opcionais **[design]**: `WEBHOOK_POLL_INTERVAL_MS`, `WEBHOOK_BATCH_SIZE`. |
| `src/modules/orders/order.status.ts` | Fonte de verdade dos status (`OrderStatus` do Prisma) usada para validar `events`. |
| `src/shared/logger/index.ts` | Reuso do Pino; ampliar `redactPaths`. |
| `package.json` | Novo script `"worker"` (ex.: `tsx src/worker.ts`, seguindo os scripts existentes) [09:11 Larissa]. |

Estrutura do módulo: `src/modules/webhooks/{webhook.controller,webhook.service,webhook.repository,webhook.routes,webhook.schemas,webhook.errors,webhook.publisher,webhook.processor}.ts`.

## 11. Dependências e compatibilidade

- Sem nova infra: MySQL, Prisma, Express, Zod, Pino existentes.
- HTTP do worker: `fetch` nativo do Node ou biblioteca já presente **[design]**; HMAC via `node:crypto`.
- Compatibilidade: nenhuma mudança nos contratos atuais. `changeStatus` ganha mais escritas na mesma transação.
- Ordering garantido apenas por `order_id` com single‑worker [09:13 Larissa].

## 12. Critérios de aceite técnicos

1. Mudança de status com webhook assinante cria exatamente 1 linha de outbox por endpoint, na mesma transação; falha forçada na outbox desfaz a mudança.
2. Status não assinado ou customer sem webhook não cria linha.
3. Evento entregue em < 10 s com cliente saudável.
4. Cliente que responde 500 gera retries nos atrasos configurados; após esgotar, linha em `webhook_dead_letter`.
5. `X-Signature` confere com HMAC-SHA256 do corpo usando a secret do endpoint; após rotação, ambas valem por 24 h.
6. Replay exige ADMIN; OPERATOR recebe 403; o log contém o `userId`.
7. URL `http://` é recusada com `WEBHOOK_INVALID_URL`; payload > 64 KB gera `WEBHOOK_PAYLOAD_TOO_LARGE`.
8. API continua respondendo com o worker parado.

## 13. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Transação de `changeStatus` mais longa | Insert simples e indexado; medir antes do deploy |
| Linhas presas em `PROCESSING` | Recuperação no start do worker |
| Secret exposta em log/DB | Redação no Pino; revisão da Sofia (≥ 2 dias úteis) [09:46 Sofia]; Q-04 em aberto |
| Interpretação errada de "5 tentativas" | Constante única; validar na revisão (Q-01) |
| Crescimento da outbox | Índices; arquivamento em fase futura [09:08 Diego] |
