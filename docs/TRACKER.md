# TRACKER — Rastreabilidade

Cada linha liga um item dos documentos à sua origem. Itens marcados **[design]** nos documentos (detalhes de implementação sem decisão explícita na reunião) não têm linha de TRANSCRICAO e estão listados na seção final.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Restrição | Três clientes B2B pedem notificação em tempo real; hoje fazem polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Restrição | Atlas pode migrar para concorrente se não entregar até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | Prazo pedido pela Atlas: fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | "Tempo real" = abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-04 | docs/PRD.md | Restrição | Webhook somente outbound (plataforma → cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com url, status e secret gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01b | docs/PRD.md | Requisito Funcional | customerId vem no body/path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status, aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Evento na outbox na mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Entrega assinada com headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry 1m/5m/30m/2h/12h e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay de DLQ restrito a ADMIN e logado | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace de 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | URL deve ser https (validação Zod) | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64 KB com erro | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança de ≥ 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Decisão | Meta de latência < 10 s com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Decisão | Janela de retry ≈ 15 h | TRANSCRICAO | [09:17] Diego |
| PRD-OOS-01 | docs/PRD.md | Restrição | Fora de escopo: e-mail de alerta | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Restrição | Fora de escopo: rate limiting de saída (observar) | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-03 | docs/PRD.md | Restrição | Fora de escopo: dashboard visual | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Restrição | Fora de escopo: arquivamento de entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Restrição | Sem ordering global / múltiplos workers | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-06 | docs/PRD.md | Trade-off | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-07 | docs/PRD.md | Trade-off | Retry indefinido descartado | TRANSCRICAO | [09:15] Diego |
| PRD-DEP-01 | docs/PRD.md | Restrição | Documentação no portal de desenvolvedor pelo PM | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-02 | docs/PRD.md | Restrição | Confirmar prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-RSK-01 | docs/PRD.md | Restrição | Prazo estimado em 3 sprints com revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-RSK-02 | docs/PRD.md | Restrição | Cliente já vazou secret em log próprio | TRANSCRICAO | [09:22] Diego |
| PRD-TST-01 | docs/PRD.md | Restrição | Testes de integração seguem padrão existente (Vitest) | CODIGO | tests/orders.test.ts |
| RFC-ALT-01 | docs/RFC.md | Trade-off | HTTP síncrono descartado (trava transação) | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado (nova infra) | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Trigger MySQL descartado (não notifica processo externo) | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Retry indefinido descartado | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | docs/RFC.md | Trade-off | 3 tentativas descartado (cobre só ~30 min) | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Trade-off | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| RFC-OQ-01 | docs/RFC.md | Restrição | Q-01: contagem de "5 tentativas" ambígua | TRANSCRICAO | [09:17] Diego |
| RFC-OQ-02 | docs/RFC.md | Restrição | Q-02: rate limiting adiado | TRANSCRICAO | [09:39] Larissa |
| RFC-OQ-03 | docs/RFC.md | Restrição | Q-03: assinatura durante grace não especificada | TRANSCRICAO | [09:21] Sofia |
| RFC-OQ-04 | docs/RFC.md | Restrição | Q-04: armazenamento da secret não discutido; revisão da Sofia | TRANSCRICAO | [09:46] Sofia |
| RFC-OQ-05 | docs/RFC.md | Restrição | Q-05: e-mail adiado | TRANSCRICAO | [09:37] Larissa |
| RFC-OQ-06 | docs/RFC.md | Restrição | Q-06: ordering com múltiplos workers adiado | TRANSCRICAO | [09:13] Diego |
| RFC-OQ-07 | docs/RFC.md | Restrição | Q-07: evento de criação (PENDING) não discutido; hook é só em changeStatus | TRANSCRICAO | [09:40] Bruno |
| RFC-OQ-08 | docs/RFC.md | Restrição | Q-08: retenção de linhas entregues fora do escopo | TRANSCRICAO | [09:08] Diego |
| RFC-IMP-01 | docs/RFC.md | Restrição | Alteração em changeStatus dentro do OrderService | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMP-02 | docs/RFC.md | Restrição | Novas rotas registradas no router da API | CODIGO | src/routes/index.ts |
| RFC-IMP-03 | docs/RFC.md | Restrição | Novas tabelas no schema Prisma | CODIGO | prisma/schema.prisma |
| FDD-FLX-01 | docs/FDD.md | Decisão | publishWebhookEvent(tx, order, from, to) dentro da transação | TRANSCRICAO | [09:41] Bruno |
| FDD-FLX-02 | docs/FDD.md | Decisão | Outbox falha → rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-03 | docs/FDD.md | Decisão | Worker em polling de 2 s, mais antigos primeiro, batch pequeno | TRANSCRICAO | [09:09] Diego |
| FDD-FLX-04 | docs/FDD.md | Decisão | Ordenação por created_at e order_id com single-worker | TRANSCRICAO | [09:12] Diego |
| FDD-FLX-05 | docs/FDD.md | Decisão | DLQ em webhook_dead_letter com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-06 | docs/FDD.md | Decisão | Replay recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-07 | docs/FDD.md | Decisão | Payload snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| FDD-FLX-08 | docs/FDD.md | Decisão | event_id UUID gerado na entrada da outbox | TRANSCRICAO | [09:25] Diego |
| FDD-DAT-01 | docs/FDD.md | Decisão | Status da outbox pendente/processando/falhou/entregue; índices status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DAT-02 | docs/FDD.md | Decisão | Config de webhook: url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DAT-03 | docs/FDD.md | Decisão | IDs em UUID | TRANSCRICAO | [09:51] Larissa |
| FDD-DAT-04 | docs/FDD.md | Restrição | Models seguem convenção UUID Char(36) e @@map snake_case | CODIGO | prisma/schema.prisma |
| FDD-CTR-01 | docs/FDD.md | Decisão | POST /webhooks | TRANSCRICAO | [09:31] Marcos |
| FDD-CTR-02 | docs/FDD.md | Decisão | GET /webhooks por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-03 | docs/FDD.md | Decisão | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-04 | docs/FDD.md | Decisão | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-05 | docs/FDD.md | Decisão | Endpoint de rotação de secret (rota é design) | TRANSCRICAO | [09:21] Sofia |
| FDD-CTR-06 | docs/FDD.md | Decisão | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CTR-07 | docs/FDD.md | Decisão | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CTR-08 | docs/FDD.md | Decisão | Payload: event_id, event_type, timestamp, order_id, order_number, from/to, customer_id, total_cents, sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CTR-09 | docs/FDD.md | Decisão | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CTR-10 | docs/FDD.md | Decisão | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CTR-11 | docs/FDD.md | Restrição | Prefixo /api/v1 e formato de erro `{error:{code,message,details}}` | CODIGO | src/app.ts |
| FDD-CTR-12 | docs/FDD.md | Restrição | Formato do erro serializado pelo middleware | CODIGO | src/middlewares/error.middleware.ts |
| FDD-ERR-01 | docs/FDD.md | Decisão | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Decisão | Prefixo WEBHOOK_ para tudo do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | docs/FDD.md | Requisito Não Funcional | WEBHOOK_PAYLOAD_TOO_LARGE para > 64 KB | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-04 | docs/FDD.md | Restrição | Erros estendem AppError e subclasses HTTP | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-05 | docs/FDD.md | Restrição | Estrutura base de AppError (statusCode, errorCode, details) | CODIGO | src/shared/errors/app-error.ts |
| FDD-RES-01 | docs/FDD.md | Decisão | Timeout 10 s trata como falha e retry | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Decisão | Backoff 1m/5m/30m/2h/12h, 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-03 | docs/FDD.md | Decisão | Payload grande: erro, sem truncar | TRANSCRICAO | [09:23] Sofia |
| FDD-RES-04 | docs/FDD.md | Decisão | Worker não deve rodar na API (restart perde worker) | TRANSCRICAO | [09:11] Diego |
| FDD-OBS-01 | docs/FDD.md | Requisito Não Funcional | Replay loga quem executou (auditoria) | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-02 | docs/FDD.md | Decisão | Pino como logger único | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-03 | docs/FDD.md | Restrição | Lista redactPaths do logger a ampliar com secret | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-04 | docs/FDD.md | Restrição | Request id disponível via request-logger | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Restrição | changeStatus usa $transaction com TxClient | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Restrição | authenticate e requireRole | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-03 | docs/FDD.md | Restrição | validate() com schemas Zod | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Restrição | Registro de controllers e rotas | CODIGO | src/routes/index.ts |
| FDD-INT-05 | docs/FDD.md | Restrição | buildControllers monta repository/service/controller | CODIGO | src/app.ts |
| FDD-INT-06 | docs/FDD.md | Restrição | Modelo de bootstrap e shutdown para o worker | CODIGO | src/server.ts |
| FDD-INT-07 | docs/FDD.md | Restrição | createPrismaClient para instância própria do worker | CODIGO | src/config/database.ts |
| FDD-INT-08 | docs/FDD.md | Restrição | Validação de env com Zod | CODIGO | src/config/env.ts |
| FDD-INT-09 | docs/FDD.md | Restrição | OrderStatus fonte dos status válidos | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-10 | docs/FDD.md | Decisão | (a criar) src/worker.ts e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-11 | docs/FDD.md | Decisão | Lógica em webhook.worker.ts / webhook.processor.ts | TRANSCRICAO | [09:28] Bruno |
| FDD-INT-12 | docs/FDD.md | Decisão | PrismaClient separado no worker, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| FDD-INT-13 | docs/FDD.md | Decisão | Módulo src/modules/webhooks com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| FDD-INT-14 | docs/FDD.md | Decisão | CRUD qualquer role autenticada; replay ADMIN com requireRole | TRANSCRICAO | [09:36] Larissa |
| FDD-INT-15 | docs/FDD.md | Decisão | Middleware de erro já trata AppError, Zod e Prisma | TRANSCRICAO | [09:29] Bruno |
| FDD-SEC-01 | docs/FDD.md | Decisão | HMAC-SHA256 sobre o corpo | TRANSCRICAO | [09:22] Sofia |
| FDD-SEC-02 | docs/FDD.md | Decisão | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| FDD-SEC-03 | docs/FDD.md | Decisão | X-Timestamp para detecção de replay pelo cliente | TRANSCRICAO | [09:44] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL, transacional, snapshot, UUID, filtro na inserção | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Alternativas: síncrono, Redis Streams, trigger | TRANSCRICAO | [09:07] Diego |
| ADR-001-COD | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Transação existente de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | Backoff 1m/5m/30m/2h/12h, 5 tentativas, DLQ separada, replay admin | TRANSCRICAO | [09:17] Larissa |
| ADR-002-CTX | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Restrição | Cliente com 2 h de manutenção planejada | TRANSCRICAO | [09:16] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-003-TLS | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Requisito Não Funcional | HTTPS obrigatório | TRANSCRICAO | [09:23] Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-004-ORD | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Restrição | Ordering só por order_id e single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-005 | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Worker separado, polling 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-005-COD | docs/adrs/ADR-005-worker-separado-em-polling.md | Restrição | Entry-point existente src/server.ts como modelo | CODIGO | src/server.ts |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-COD1 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | AppError como base de erros | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-COD2 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | Logger Pino com redactPaths | CODIGO | src/shared/logger/index.ts |
| ADR-007 | docs/adrs/ADR-007-contrato-de-envio-payload-headers-limites.md | Decisão | Payload enxuto, headers, timeout 10 s, limite 64 KB | TRANSCRICAO | [09:43] Diego |

## Itens marcados [design] (sem origem direta na transcrição)

Detalhes de implementação propostos pelo autor, derivados dos itens acima e sujeitos à revisão: tabelas `webhook_endpoints` e `webhook_deliveries` (derivadas de [09:21] Bruno/Sofia e [09:34] Marcos); rota `POST /webhooks/:id/rotate-secret`; tamanho do batch; recuperação de linhas `PROCESSING`; formato hex da assinatura e prefixo `whsec_`; métricas e tracing por `event_id`; variáveis de ambiente do worker; interpretação "1 envio + 5 retries" (RFC Q-01); duas assinaturas durante o grace (Q-03). Ver marcações no [FDD](FDD.md).
