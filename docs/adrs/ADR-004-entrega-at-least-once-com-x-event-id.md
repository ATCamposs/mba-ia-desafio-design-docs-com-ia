# ADR-004: Garantia de entrega at-least-once com `X-Event-Id`

- **Status:** Aceita (decidida na reunião técnica; ver `TRANSCRICAO.md` [09:26] Larissa)
- **Data do registro:** 2026-10-26
- **Decisores:** Diego, Larissa, Sofia (levantou a transferência de responsabilidade), Marcos (assumiu a documentação para clientes)
- **Documentos relacionados:** [RFC](../RFC.md), [FDD](../FDD.md) (seções "Contratos públicos" e "Estratégias de resiliência"), [ADR-002](ADR-002-retry-com-backoff-e-dlq.md), [ADR-003](ADR-003-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

Com retry (ADR-002) e worker que pode cair entre a chamada HTTP e a marcação da linha como entregue, o mesmo evento pode chegar mais de uma vez ao cliente. É preciso escolher a semântica de entrega e dar ao cliente um jeito de reconhecer duplicatas.

## Decisão

1. **A semântica de entrega é at-least-once**: o cliente pode receber o mesmo evento duas vezes e "tem que estar preparado" ([09:24] Diego).
2. Cada evento recebe um **UUID gerado quando entra na outbox**, enviado no header **`X-Event-Id`**. É único por evento ([09:25] Diego).
3. O cliente **deduplica por `event_id` do lado dele** ([09:25] Diego).
4. O mesmo identificador vai também no campo `event_id` do corpo JSON ([09:43] Diego).
5. Um replay manual da DLQ (ADR-002) reenvia o **mesmo `event_id`**, para que o cliente que já deduplica não processe de novo. *(Derivado: a reunião disse que o replay "recoloca na outbox como pendente" ([09:18] Diego), sem tratar do identificador.)*
6. Marcos documentará o comportamento "bem destacado no portal de desenvolvedor" ([09:26] Marcos).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | "Exigiria coordenação dos dois lados e fica muito mais complexo" ([09:25] Diego). |
| **At-most-once** (sem retry, plausível) | Perderia notificações sempre que o cliente estivesse indisponível, o oposto do objetivo da feature ([09:00] Marcos) e da política de retry ([09:17] Larissa). |
| **Deduplicar no lado da plataforma** (plausível) | A plataforma não sabe se o cliente processou uma entrega cuja resposta se perdeu; só o cliente consegue decidir. Por isso o `event_id` vai ao cliente. |

## Consequências

**Positivas**

- Simplicidade: sem coordenação distribuída com o cliente.
- É o padrão de mercado: "Stripe faz assim, GitHub faz assim" ([09:25] Diego). Resolve "99% dos casos" ([09:25] Diego).
- O worker pode ser tolerante a falhas: se cair depois do HTTP e antes de marcar `DELIVERED`, a linha é reenviada sem perder o evento.

**Negativas / trade-offs**

- **Responsabilidade transferida ao cliente**, como apontou Sofia ([09:25] Sofia). Cliente que não deduplica processa notificações repetidas.
- A documentação do portal vira parte do contrato; se for fraca, a garantia na prática falha.
- Duplicatas ocorrem em pelo menos três situações: timeout do cliente depois de processar, queda do worker entre a resposta e o `UPDATE`, e replay manual.
- Não há ordering global (ver ADR-005); com retry, um evento antigo pode chegar depois de um mais novo do mesmo pedido.

## Referências ao código

- `package.json`: `uuid` 11.0.3 já é dependência; `src/middlewares/request-logger.middleware.ts` usa `uuid` v4 para `requestId`, e o mesmo gerador serve para `event_id`.
