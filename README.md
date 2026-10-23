# Design Docs — Webhooks de Notificação de Pedidos

## Sobre o desafio

Transformar a transcrição de uma reunião técnica (`TRANSCRICAO.md`) e o código de um Order Management System (Node.js + TypeScript + Prisma/MySQL) em um pacote de design docs para a feature de webhooks de notificação de pedidos. A entrega é só documental: o código em `src/`, `prisma/` e `tests/` não foi alterado. Tudo tem que ser rastreável à transcrição ou ao código.

## Ferramentas de IA utilizadas

- **Claude Code (Sonnet 5.5):** leitura do código e da transcrição, rascunho de ADRs, RFC, FDD, PRD, tracker e README.
- **Revisão manual:** conferência de cada item contra a transcrição e dos caminhos de arquivo contra o repositório.

## Workflow adotado

1. Fork, clone e leitura da transcrição inteira e dos arquivos-chave (`order.service.ts`, `schema.prisma`, erros, middlewares, `server.ts`).
2. ADRs primeiro (7), com timestamps da reunião em cada decisão.
3. RFC (proposta, alternativas, questões em aberto) apontando para os ADRs.
4. FDD (fluxos, contratos, erros, integração com o código).
5. PRD por último, consolidando.
6. TRACKER varrendo os documentos prontos; README no fim.
7. Checagem final: caminhos de arquivo existem, links entre docs funcionam, nenhum item descartado virou requisito.

## Prompts customizados

Prompt de extração (filtragem do que entra e do que não entra):

```text
Leia TRANSCRICAO.md. Classifique cada ponto em: (a) decisão fechada, (b) requisito
funcional, (c) requisito não funcional, (d) descartado, (e) adiado/em aberto.
Para cada item cite [hh:mm] Falante. Não infira nada sem fala que sustente.
Liste separadamente ambiguidades e contradições da transcrição.
```

Prompt de escrita de ADR:

```text
Escreva o ADR-00X em MADR (Status, Contexto, Decisão, Alternativas Consideradas,
Consequências positivas e negativas). Cada afirmação deve vir com [hh:mm] Falante.
Se um detalhe for suposição sua, marque "[design]" em vez de apresentá-lo como decisão.
Cite arquivos reais do repo apenas se eu os abri nesta sessão.
```

## Iterações e ajustes

Quatro ciclos principais:

1. **Ambiguidade das tentativas:** a primeira versão tratou "5 tentativas" como certa. Ao somar os intervalos (1m+5m+30m+2h+12h ≈ 14h36, "quase 15 h") ficou claro que a fala é ambígua; virou a questão em aberto Q-01 e uma constante única no FDD.
2. **Itens inventados:** a IA incluiu coisas sem origem (tabelas de entrega, rota de rotação, métricas). Passei a marcá-las como `[design]` no FDD e a listá-las à parte no tracker.
3. **Rotas e prefixo:** a transcrição cita `/admin/webhooks/...` sem prefixo; no código as rotas ficam sob `/api/v1` (`src/app.ts`). Ajustei o FDD.
4. **Lacunas de segurança:** a IA não notou que a `secret` precisa ser recuperável para assinar, nem que o logger só redige `password`/`token`. Virou Q-04 e ajuste de `redactPaths` no FDD. Também corrigi a ligação errada da transcrição em [09:35] (o "endpoint admin" é o replay, não o histórico de entregas).

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](docs/PRD.md) — por que e o quê
2. [`docs/RFC.md`](docs/RFC.md) — proposta e questões em aberto
3. [`docs/adrs/`](docs/adrs/) — ADR-001 a ADR-007
4. [`docs/FDD.md`](docs/FDD.md) — como construir
5. [`docs/TRACKER.md`](docs/TRACKER.md) — origem de cada item
