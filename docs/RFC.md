# RFC-001: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Gabriel Lima |
| **Status** | Em revisão |
| **Data** | 2026-10-22 |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança) |
| **Origem** | Reunião técnica de quinta-feira, 09:00 (`TRANSCRICAO.md`) |

## 1. Resumo executivo (TL;DR)

Vamos notificar clientes B2B sobre mudanças de status de pedido via **webhooks HTTPS assinados**. A mudança de status grava um evento numa tabela **outbox no MySQL**, na mesma transação do `OrderService.changeStatus`. Um **worker separado**, em polling de 2 s, entrega os eventos com **retry exponencial (1m/5m/30m/2h/12h)** e, esgotadas as tentativas, move para uma **DLQ** com replay manual por ADMIN. A entrega é **at-least-once**, com `X-Event-Id` para deduplicação, e a assinatura é **HMAC-SHA256 com secret por endpoint**. Pedimos revisão da abordagem e das questões em aberto (seção 6) antes de iniciar a implementação.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo consultam `GET /orders` periodicamente para saber se um pedido mudou, o que torna a integração lenta e cara; a Atlas ameaça migrar para um concorrente se não houver entrega até o fim do trimestre [09:00 Marcos]. Para eles, "tempo real" é menos de 10 segundos [09:02 Marcos]. A fluxo é somente de saída: a plataforma envia, o cliente não envia nada [09:02 Marcos].

A aplicação hoje não tem eventos, filas nem webhooks. A mudança de status é transacional e pesada (`orders`, `order_status_history`, estoque), o que impede chamadas HTTP no meio dela [09:04 Bruno].

## 3. Proposta técnica

Visão geral (detalhes de implementação em [FDD](FDD.md)):

1. **Registro atômico do evento.** `changeStatus` chama `publishWebhookEvent(tx, order, from, to)` dentro da própria transação. Para cada webhook ativo do cliente que assina o novo status, é inserida uma linha em `webhook_outbox` com o payload já renderizado ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).
2. **Entrega assíncrona.** Um worker em processo próprio (`src/worker.ts`) consulta a outbox a cada 2 s e faz o POST ao cliente com timeout de 10 s ([ADR-005](adrs/ADR-005-worker-separado-em-polling.md)).
3. **Falha e recuperação.** Falhas entram em backoff até 5 tentativas; depois vão para `webhook_dead_letter`, de onde um ADMIN pode reenfileirar ([ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md)).
4. **Segurança e semântica de entrega.** Corpo assinado com HMAC-SHA256, secret única por endpoint, rotação com grace de 24 h ([ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)); at-least-once com `X-Event-Id` ([ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)); payload enxuto e headers padronizados ([ADR-007](adrs/ADR-007-contrato-de-envio-payload-headers-limites.md)).
5. **API de gestão.** CRUD de webhooks, rotação de secret, histórico de entregas e replay de DLQ, seguindo os padrões do projeto ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)).

## 4. Alternativas consideradas

| # | Alternativa | Por que foi descartada (trade-off) |
|---|---|---|
| A1 | **HTTP síncrono no `changeStatus`** | Simples, mas um cliente lento trava a transação e o rollback por indisponibilidade do cliente seria inaceitável [09:04 Bruno]. |
| A2 | **Redis Streams / fila dedicada** | Boa vazão, mas exige subir e operar nova infra; overengineering para um time pequeno [09:07 Larissa, Diego]. |
| A3 | **Trigger MySQL para notificar o worker** | Seria mais reativo, mas trigger só executa SQL e não acorda processo externo; polling de 2 s já atende [09:09 Diego]. |
| A4 | **Retry indefinido** | Nunca perde evento, porém deixa evento pendurado para sempre se o cliente sumiu [09:15 Diego]. |
| A5 | **3 tentativas** | Mais agressivo, mas cobre só ~30 min e não resiste a manutenções de 2 h [09:16 Diego]. |
| A6 | **Exactly-once** | Garantia mais forte, mas exige coordenação dos dois lados e complexidade alta [09:25 Diego]. |

## 5. Impacto e riscos

**Impacto no sistema existente:** alteração em `src/modules/orders/order.service.ts` (uma chamada dentro de `changeStatus`), nova entry-point `src/worker.ts` (a criar), novas tabelas no `prisma/schema.prisma`, novas rotas em `src/routes/index.ts`. Nenhuma mudança no contrato público de `/orders`.

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Falha na inserção da outbox derruba a mudança de status | Baixa | Alto | Comportamento intencional (rollback) para não perder evento [09:40 Bruno]; testes de integração. |
| Worker único é ponto único de falha | Média | Médio | Processo separado com restart automático; alerta de backlog (FDD). |
| Vazamento de secret | Média | Alto | Secret por endpoint, rotação, redação em log, revisão da Sofia [09:21 Sofia, 09:46 Sofia]. |
| Cliente recebe duplicata | Alta | Baixo | At-least-once documentado, `X-Event-Id` [09:25 Diego]. |
| Prazo de 3 sprints | Média | Médio | Escopo fechado; replay, rate limit e e-mail fora [09:46 Larissa]. |

## 6. Questões em aberto

| ID | Questão | Situação na reunião |
|---|---|---|
| Q-01 | A "5 tentativas" inclui a primeira? Há cinco intervalos (1m…12h) e o total de ~15 h (1m+5m+30m+2h+12h ≈ 14h36) sugere 1 envio inicial + 5 retries. | Ambíguo [09:15 Diego, 09:17 Diego]. Proposta: inicial + 5 retries. |
| Q-02 | **Rate limiting de saída** por cliente (ex.: 50 pedidos mudando em 1 min). | Adiado: "observar e decidir depois" [09:39 Larissa]. |
| Q-03 | Como assinar durante o grace period de 24 h com duas secrets válidas? | Não discutido [09:21 Sofia]. Proposta no FDD: enviar as duas assinaturas em `X-Signature`. |
| Q-04 | Armazenamento da secret (criptografia em repouso, chave). | Não discutido; revisão de segurança da Sofia [09:46 Sofia]. |
| Q-05 | Notificação por e-mail ao cliente após falhas. | Adiado para a próxima fase [09:37 Larissa]. |
| Q-06 | Ordenação com múltiplos workers (particionar por `order_id` ou lock pessimista). | Adiado: "problema do futuro" [09:13 Diego]. |
| Q-07 | O evento inicial de criação do pedido (PENDING, `fromStatus` nulo) gera webhook? | Não discutido; a reunião só trata `changeStatus` [09:40 Bruno]. Proposta: não gera. |
| Q-08 | Arquivamento/retenção de linhas entregues (≈30 dias). | Fora do escopo desta feature [09:08 Diego]. |

## 7. Decisões relacionadas

- [ADR-001 — Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry com backoff e DLQ](adrs/ADR-002-retry-backoff-e-dlq.md)
- [ADR-003 — HMAC-SHA256 com secret por endpoint](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — At-least-once com X-Event-Id](adrs/ADR-004-at-least-once-com-x-event-id.md)
- [ADR-005 — Worker separado em polling](adrs/ADR-005-worker-separado-em-polling.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Contrato de envio](adrs/ADR-007-contrato-de-envio-payload-headers-limites.md)

Documentos complementares: [PRD](PRD.md), [FDD](FDD.md), [TRACKER](TRACKER.md).
