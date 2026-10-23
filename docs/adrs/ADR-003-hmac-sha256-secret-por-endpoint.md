# ADR-003: Autenticação HMAC-SHA256 com secret por endpoint e rotação

- **Status:** Aceito
- **Data:** 2026-10-22
- **Decisores:** Sofia, Diego, Larissa
- **Relacionados:** [ADR-004](ADR-004-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-contrato-de-envio-payload-headers-limites.md)

## Contexto

Os eventos carregam dados de pedidos para um endpoint fora da nossa infra. O cliente precisa validar que a requisição veio de nós e que o payload não foi adulterado [09:19 Sofia]. Já houve cliente que vazou secret em log da própria aplicação [09:22 Diego].

## Decisão

- Assinar o **corpo do request** com **HMAC-SHA256** e enviar no header `X-Signature` [09:20 Sofia, 09:22 Sofia].
- **Uma secret única por endpoint de webhook**, nunca global [09:21 Sofia]. A secret é gerada pela plataforma e devolvida na criação [09:31 Marcos].
- **Rotação** disponível via API; ao rotacionar, a secret antiga continua válida por **24 h** (grace period) e depois expira [09:21 Sofia].
- URL do webhook deve ser **HTTPS**; `http` é recusado por validação Zod [09:23 Sofia].
- Sofia revisa o código de segurança (HMAC e geração de secret) por pelo menos 2 dias úteis antes do deploy [09:46 Sofia].

## Alternativas Consideradas

1. **Secret global da plataforma** — descartada: vazou uma, vazaram todas [09:21 Sofia].
2. **Sem assinatura, só HTTPS** — plausível, mas não atende: não prova origem nem integridade do payload [09:19 Sofia].
3. **Outros algoritmos de assinatura** — HMAC-SHA256 foi escolhido por ser padrão de mercado com biblioteca em todo cliente sério [09:20 Sofia]. Alternativas específicas não foram discutidas.

## Consequências

**Positivas**
- Isolamento de dano: o vazamento afeta um endpoint só.
- O cliente migra sem corte graças ao grace period.

**Negativas / trade-offs**
- A plataforma precisa guardar a secret de forma recuperável (não apenas hash) para assinar; armazenamento seguro e criptografia em repouso não foram detalhados na reunião (RFC, Q-04).
- Durante o grace period existem duas secrets ativas; como assinar nesse intervalo não foi especificado (RFC, Q-03).
- O `X-Timestamp` não entra na assinatura (a decisão foi "HMAC sobre o corpo"), então a proteção contra replay é opcional e fica do lado do cliente [09:44 Diego].
