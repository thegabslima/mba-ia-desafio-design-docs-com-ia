# ADR-001: Padrão Outbox no MySQL existente

- **Status:** Aceito
- **Data:** 2026-10-22
- **Decisores:** Larissa, Diego, Bruno (reunião de design, quinta-feira 09:00)
- **Relacionados:** [ADR-005](ADR-005-worker-separado-em-polling.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) querem ser notificados quando o status de um pedido muda [09:00 Marcos]. Hoje a mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que atualiza `orders`, insere em `order_status_history` e mexe no estoque. Fazer HTTP dentro dessa transação travaria mudanças de outros pedidos quando um cliente estiver lento, e não haveria o que fazer com o rollback se o cliente estivesse fora do ar [09:04 Bruno].

## Decisão

Adotar o padrão **Outbox no MySQL existente** [09:08 Larissa]:

- Tabela `webhook_outbox` recebe uma linha por (evento, endpoint de webhook) **na mesma transação** que atualiza `orders` e `order_status_history` [09:06 Diego, 09:40 Bruno]. Falha ao inserir na outbox causa rollback da mudança de status.
- Status da linha: `PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`; índices em `status` e `created_at` [09:08 Diego].
- Chave primária UUID, como no resto do schema Prisma [09:51 Larissa].
- O payload é gravado **renderizado (snapshot)** no momento da inserção [09:52 Larissa].
- O filtro de status assinados pelo webhook é aplicado **na inserção**: se nenhum webhook quer aquele status, nenhuma linha é criada [09:34 Bruno].
- Arquivamento de linhas entregues (30 dias) está fora do escopo [09:08 Diego].

## Alternativas Consideradas

1. **Chamada HTTP síncrona dentro do `changeStatus`** — descartada: bloqueia a transação e acopla a disponibilidade do cliente à nossa [09:04 Bruno, 09:06 Diego].
2. **Redis Streams ou fila similar** — descartada: exige nova infra; "Redis Cluster pra isso é overengineering" para um time pequeno [09:07 Larissa, Diego].
3. **Gatilho (trigger) de banco para notificar o worker** — descartada: trigger só executa SQL, não avisa processo externo [09:09 Diego].
4. **Guardar só `order_id` e renderizar o payload no envio** — descartada: o evento deixaria de refletir o estado do momento da mudança [09:52 Larissa].

## Consequências

**Positivas**
- Atomicidade: se a transação principal commitou, o evento existe; se deu rollback, some junto [09:06 Diego].
- Nenhuma infra nova; reaproveita MySQL e Prisma.
- Payload imutável: reprocessamentos enviam o mesmo conteúdo.

**Negativas / trade-offs**
- A transação de `changeStatus`, já pesada, ganha mais escritas (uma por webhook interessado).
- A outbox cresce no MySQL principal; retenção fica para fase futura.
- Latência depende de polling (ver ADR-005).
