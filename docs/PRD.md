# PRD — Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e contexto

Feature que notifica clientes B2B, via webhook HTTPS, sempre que o status de um pedido muda no Order Management System. Hoje a plataforma não tem nenhum mecanismo de notificação externa. Detalhes técnicos: [RFC](RFC.md), [FDD](FDD.md), [ADRs](adrs/).

## 2. Problema e motivação

Três clientes — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para ser avisados em tempo real; hoje consultam `GET /orders` repetidamente, o que deixa a integração lenta e cara [09:00 Marcos]. A Atlas indicou que pode migrar para um concorrente se não houver entrega até o fim do trimestre [09:00 Marcos]; o prazo pedido é fim de novembro [09:45 Marcos].

## 3. Público-alvo e cenários de uso

- **Cliente B2B (sistema integrador):** cadastra um endpoint, escolhe quais status quer ouvir e recebe um POST a cada mudança.
- **Operador/ADMIN da plataforma:** reprocessa eventos que falharam definitivamente (somente ADMIN) [09:36 Sofia].
- **PM (Marcos):** documenta a integração no portal de desenvolvedor [09:26, 09:40 Marcos].

Cenários: (1) "só quero saber quando vira SHIPPED e DELIVERED" [09:33 Marcos]; (2) cliente fora do ar por manutenção de 2 h volta e recebe o que perdeu pelo retry; (3) cliente consulta os últimos 100 envios para depurar [09:34 Marcos]; (4) cliente troca a secret sem indisponibilidade [09:21 Sofia].

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
|---|---|---|
| Notificação percebida como "tempo real" | Tempo entre commit da mudança e primeira tentativa de entrega | < 10 s (limite do cliente) [09:02 Marcos]; polling de 2 s [09:10 Larissa] |
| Não perder eventos | Mudanças de status com evento correspondente na outbox (quando há assinante) | 100% (atomicidade) [09:40 Bruno] |
| Atender os clientes no prazo | Clientes com webhook ativo | 3 (Atlas, MaxDistribuição, Nova Cargo) até o fim de novembro [09:45 Marcos] |
| Resiliência a indisponibilidade do cliente | Janela coberta por retry | ≈ 15 h [09:17 Diego] |

Meta de percentual de entregas bem-sucedidas na primeira tentativa não foi definida na reunião; a ser calibrada após medição.

## 5. Escopo

### Incluso
Cadastro/edição/remoção/listagem de webhooks; filtro de status; geração e rotação de secret; entrega assinada com retry e DLQ; histórico de entregas; replay admin.

### Fora de escopo (descartado ou adiado)
- **E-mail de alerta ao cliente** quando o webhook falha repetidamente — fora desta fase, talvez na próxima [09:37 Larissa].
- **Rate limiting de saída** — "observar e decidir depois" [09:39 Larissa].
- **Dashboard visual** para o cliente — projeto separado do time de frontend [09:40 Larissa].
- **Arquivamento de eventos entregues** (≈30 dias) — fora desta feature [09:08 Diego].
- **Ordenação global / múltiplos workers** — limitação conhecida [09:13 Larissa].
- **Garantia exactly-once** e **retry indefinido** — descartados [09:25 Diego, 09:15 Diego].
- **Webhooks de entrada** (cliente enviando para nós) [09:02 Marcos].

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| RF-01 | Cadastrar webhook (`url`, lista de status; secret gerada e devolvida na criação; `customerId` informado na requisição) | [09:31 Marcos, 09:32 Larissa] |
| RF-02 | Editar webhook | [09:33 Bruno] |
| RF-03 | Remover webhook | [09:33 Bruno] |
| RF-04 | Listar webhooks de um customer | [09:33 Bruno] |
| RF-05 | Filtrar eventos por status assinado, aplicado na inserção da outbox | [09:33 Marcos, 09:34 Bruno] |
| RF-06 | Registrar evento na outbox na mesma transação da mudança de status | [09:40 Bruno] |
| RF-07 | Entregar via POST assinado (HMAC-SHA256) com headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` | [09:44 Diego, Sofia] |
| RF-08 | Retry com backoff 1m/5m/30m/2h/12h e envio à DLQ ao esgotar | [09:17 Larissa] |
| RF-09 | Consultar histórico de entregas (últimas 100) de um webhook | [09:34 Marcos] |
| RF-10 | Replay manual de item da DLQ, restrito a ADMIN e auditado em log | [09:18 Diego, 09:36 Sofia] |
| RF-11 | Rotacionar secret via API, mantendo a antiga válida por 24 h | [09:21 Sofia] |
| RF-12 | Rejeitar URL que não seja HTTPS | [09:23 Sofia] |

## 7. Requisitos não funcionais

- **Desempenho:** entrega < 10 s; polling de 2 s; timeout de 10 s por chamada [09:42 Diego].
- **Segurança:** HMAC-SHA256, secret por endpoint, TLS obrigatório, revisão de segurança de ≥ 2 dias úteis antes do deploy [09:46 Sofia].
- **Limite de payload:** 64 KB, com erro se exceder [09:24 Larissa].
- **Confiabilidade:** at-least-once; cliente deduplica por `X-Event-Id` [09:25 Diego].
- **Isolamento:** worker em processo separado da API [09:11 Diego].
- **Manutenibilidade:** padrões do projeto reaproveitados (módulo, `AppError`, Pino, prefixo `WEBHOOK_`) [09:30 Larissa].

## 8. Decisões e trade-offs principais

| Decisão | Trade-off | ADR |
|---|---|---|
| Outbox no MySQL | Sem infra nova, porém mais carga no banco principal | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Retry 5x + DLQ | Cobre ~15 h, mas depois exige replay manual | [ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md) |
| HMAC por endpoint | Isolamento de vazamento, porém gestão de secrets | [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md) |
| At-least-once | Simples, mas o cliente precisa deduplicar | [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md) |
| Worker em polling | Latência mínima de 2 s e single-worker | [ADR-005](adrs/ADR-005-worker-separado-em-polling.md) |
| Reuso de padrões | Consistência, com pequeno acoplamento no `OrderService` | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Payload enxuto | Cliente busca detalhes em `GET /orders/:id` | [ADR-007](adrs/ADR-007-contrato-de-envio-payload-headers-limites.md) |

## 9. Dependências

- Revisão de segurança da Sofia antes do deploy [09:46 Sofia].
- Portal de desenvolvedor documentado pelo PM, incluindo a deduplicação [09:26 Marcos].
- Confirmação do prazo com a Atlas [09:47 Marcos].
- Código existente: `OrderService.changeStatus`, Prisma/MySQL, `requireRole`.

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Perder o prazo de fim de novembro (Atlas pode migrar) | Média | Alto | Estimativa de 3 sprints com a revisão de segurança incluída [09:46 Larissa]; escopo enxuto |
| Vazamento de secret de cliente | Média | Alto | Secret por endpoint, rotação com grace, redação em logs [09:22 Diego] |
| Cliente processa evento duplicado | Alta | Baixo | `X-Event-Id` e documentação destacada [09:26 Marcos] |
| Cliente sobrecarregado por rajada de eventos | Baixa | Médio | Sem rate limit agora; observar e decidir depois [09:39 Larissa] |
| Falha do worker único atrasa todas as entregas | Média | Médio | Processo separado, reinício automático, alerta de backlog |

## 11. Critérios de aceitação

1. Cliente cadastra webhook HTTPS e recebe a secret uma única vez.
2. Mudança de status para um status assinado resulta em POST assinado em menos de 10 s.
3. Status não assinado não gera chamada.
4. Cliente indisponível recebe retry nos intervalos definidos; após esgotar, o evento aparece na DLQ.
5. ADMIN reprocessa item da DLQ; OPERATOR é recusado.
6. Histórico de entregas mostra sucesso/falha, payload, resposta e tempo.
7. Rotação mantém a secret antiga válida por 24 h.
8. Pedido e evento nunca ficam inconsistentes entre si.

## 12. Estratégia de testes e validação

- **Unitários:** assinatura HMAC, cálculo de backoff, filtro de eventos, validação Zod (https, 64 KB).
- **Integração (Vitest, como em `tests/orders.test.ts`):** `changeStatus` gera outbox atômica e faz rollback quando a inserção falha; worker contra servidor HTTP falso (200, 500, timeout); DLQ e replay; autorização ADMIN.
- **Segurança:** revisão da Sofia de HMAC e geração de secret.
- **Validação com clientes:** homologação com um dos três clientes antes do go-live, afim de confirmar a dedup por `X-Event-Id`.
