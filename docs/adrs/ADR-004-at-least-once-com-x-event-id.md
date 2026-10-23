# ADR-004: Entrega at-least-once com X-Event-Id para deduplicação

- **Status:** Aceito
- **Data:** 2026-10-22
- **Decisores:** Diego, Larissa, Sofia, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-retry-backoff-e-dlq.md)

## Contexto

Com outbox, retry e worker em polling, um evento pode ser entregue mais de uma vez (ex.: o cliente processou, mas a resposta estourou o timeout de 10 s). É preciso definir a garantia de entrega e como o cliente se defende de duplicatas [09:24 Diego].

## Decisão

- Garantia **at-least-once** [09:24 Diego, 09:26 Larissa].
- Cada evento recebe um **UUID gerado quando entra na outbox**, enviado no header **`X-Event-Id`**; o cliente deduplica por ele [09:25 Diego].
- A responsabilidade de dedup é do cliente e será documentada com destaque no portal de desenvolvedor [09:26 Marcos].
- Ordenação: apenas por `order_id`, e somente enquanto houver **um único worker**; não há garantia de ordering global [09:12 Diego, 09:13 Larissa].

## Alternativas Consideradas

1. **Exactly-once** — descartada: exige coordenação dos dois lados e é muito mais complexa [09:25 Diego].
2. **At-most-once** (sem retry) — plausível, não discutida; incompatível com a política de retry do ADR-002.

## Consequências

**Positivas**
- Simples e alinhada ao mercado ("Stripe faz assim, GitHub faz assim") [09:25 Diego].
- "Resolve 99% dos casos" [09:25 Diego].

**Negativas / trade-offs**
- Transfere trabalho ao cliente [09:25 Sofia].
- Com múltiplos workers a ordenação por pedido se perde; particionar por `order_id` ou usar lock pessimista ficam como evolução futura [09:13 Diego].
