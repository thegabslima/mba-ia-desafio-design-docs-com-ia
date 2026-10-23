# ADR-007: Contrato de envio — payload enxuto, headers, timeout e limite de tamanho

- **Status:** Aceito
- **Data:** 2026-10-22
- **Decisores:** Diego, Sofia, Larissa, Marcos
- **Relacionados:** [ADR-003](ADR-003-hmac-sha256-secret-por-endpoint.md), [ADR-004](ADR-004-at-least-once-com-x-event-id.md)

## Contexto

O cliente precisa receber informação suficiente para atualizar seu sistema, sem inflar o evento, e ter meios de validar origem, unicidade e frescor da chamada.

## Decisão

- **Payload JSON** com `event_id`, `event_type` (`order.status_changed`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido como `total_cents`. **Sem `items`**: o cliente consulta `GET /orders/:id` se precisar de detalhes [09:43 Diego].
- **Headers:** `X-Event-Id`, `X-Signature`, `X-Timestamp` (momento do envio, para detecção de replay pelo cliente), `X-Webhook-Id` (id do endpoint) e `Content-Type: application/json` [09:44 Diego, Sofia].
- **Timeout** da chamada HTTP: **10 s**; estouro é falha e vai para retry [09:42 Diego].
- **Limite de payload: 64 KB**; acima disso, **erro** (sem truncar) [09:23 Sofia, 09:24 Diego, Larissa].

## Alternativas Consideradas

1. **Incluir `items` no payload** — descartada: infla o evento [09:43 Diego].
2. **Truncar payloads grandes** — descartada: se o evento chegou a esse tamanho, há algo errado [09:23 Sofia].
3. **Teto maior que 64 KB** — desnecessário: "nenhum evento nosso vai chegar perto" [09:24 Diego].

## Consequências

**Positivas**
- Evento pequeno e previsível; fácil de assinar e de logar.
- `X-Webhook-Id` permite ao cliente com vários cadastros saber qual deles recebeu o envio.

**Negativas / trade-offs**
- O cliente faz uma segunda chamada para obter os itens do pedido.
- Evento acima de 64 KB falha e não é entregue (tratado como erro; ver FDD).
- Mudanças futuras no payload exigem versionamento, que não foi discutido.
