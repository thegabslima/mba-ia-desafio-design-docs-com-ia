# ADR-005: Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data:** 2026-10-22
- **Decisores:** Diego, Larissa, Bruno, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Os clientes consideram "tempo real" qualquer coisa abaixo de 10 segundos [09:02 Marcos]. Alguém precisa ler a outbox e fazer as chamadas HTTP, sem acoplar isso à vida da API [09:11 Diego].

## Decisão

- O worker faz **polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos em **batch pequeno**, processando e marcando o resultado [09:08 Diego, 09:09 Diego].
- Roda como **processo separado da API**, em nova entry-point `src/worker.ts` (arquivo novo) com script `npm run worker`, ao lado de `src/server.ts` [09:11 Larissa, Diego].
- Mesmo banco e mesma `DATABASE_URL`, mas **instância própria do `PrismaClient`** (um por processo) [09:30 Bruno].
- A lógica de processamento fica no módulo, em `src/modules/webhooks/webhook.worker.ts` (novo) ou `webhook.processor.ts` [09:28 Bruno].
- **Single-worker** por ora; a latência mínima de 2 s no pior caso foi aceita [09:10 Larissa].

## Alternativas Consideradas

1. **Trigger no MySQL para acionar o worker** — descartada: MySQL não tem LISTEN/NOTIFY e trigger só roda SQL [09:09 Diego].
2. **Worker dentro do processo da API** — descartada: se a API reinicia, perde-se o worker [09:11 Diego].
3. **Redis Streams** — descartada (ver ADR-001).

## Consequências

**Positivas**
- 2 s de polling atende com folga o requisito de < 10 s [09:09 Diego].
- Deploy e restart independentes da API.

**Negativas / trade-offs**
- Consulta periódica constante ao MySQL, mesmo sem eventos.
- Um único worker é ponto único de falha e limite de vazão; escalar exige particionamento ou lock (fora desta fase) [09:13 Diego].
- Novo processo para operar e monitorar.
