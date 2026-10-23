# ADR-002: Retry com backoff exponencial e DLQ em tabela separada

- **Status:** Aceito
- **Data:** 2026-10-22
- **Decisores:** Diego, Larissa, Bruno, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-004](ADR-004-at-least-once-com-x-event-id.md)

## Contexto

Clientes podem ficar indisponíveis por períodos longos; já houve cliente com duas horas de manutenção planejada [09:16 Diego]. É preciso uma política que não perca eventos nem deixe evento pendurado para sempre quando o cliente some [09:15 Diego].

## Decisão

- **Backoff exponencial** com intervalos de **1 min, 5 min, 30 min, 2 h e 12 h**, totalizando quase 15 horas entre a primeira falha e a última tentativa [09:17 Diego, Larissa].
- Teto de **5 tentativas**; esgotado o teto, o evento é considerado falha permanente e movido para a **DLQ** [09:15 Diego, 09:17 Larissa].
- A DLQ é a tabela **`webhook_dead_letter`**, separada da outbox, com payload, motivo da falha e timestamp [09:18 Diego].
- Reprocessamento **manual** via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente [09:18 Diego]. Exige role `ADMIN` (reuso de `requireRole`) e registra em log quem executou o replay [09:36 Sofia, Larissa].
- Timeout de 10 s por chamada conta como falha e entra no fluxo de retry [09:42 Diego].

## Alternativas Consideradas

1. **Retry indefinido com backoff** — descartada: evento fica pendurado para sempre se o cliente sumiu [09:15 Diego].
2. **3 tentativas** — descartada: cobriria só ~30 min e mataria eventos em indisponibilidades de manhã ou manutenções de 2 h [09:16 Diego].
3. **Marcar como `failed` na própria outbox** (sem tabela separada) — descartada em favor da tabela dedicada: deixa a leitura da outbox mais limpa e serve de evidência para debug e replay [09:18 Diego].
4. **Notificar o cliente por e-mail após falhas** — adiada para fase futura [09:37 Larissa].

## Consequências

**Positivas**
- Cobre janelas de indisponibilidade de até ~15 h sem intervenção.
- A outbox principal fica enxuta; a DLQ preserva evidência para auditoria e replay.

**Negativas / trade-offs**
- Cliente fora do ar por mais de ~15 h perde a entrega automática e depende de replay manual por um ADMIN.
- Mais uma tabela e um endpoint admin para manter.
- A reunião fala em "5 tentativas" e em cinco intervalos; a contagem exata (inicial + 5 retries, ou 5 no total) ficou em aberto (ver RFC, Q-01).
