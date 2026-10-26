# ADR-007: Payload em snapshot e filtro de eventos na inserção da outbox

- **Status:** Aceita (decidida na reunião técnica; ver `TRANSCRICAO.md` [09:52] Bruno e [09:34] Diego)
- **Data do registro:** 2026-10-26
- **Decisores:** Larissa, Diego, Bruno
- **Documentos relacionados:** [RFC](../RFC.md), [FDD](../FDD.md) (seções "Fluxos detalhados" e "Integração com o sistema existente"), [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Duas perguntas de desenho surgiram sobre o que entra na outbox: (a) o que se grava, o payload pronto ou só o `order_id` para renderizar na hora do envio; (b) quando se aplica o filtro de eventos que o cliente escolhe (por exemplo, "só quero SHIPPED e DELIVERED", [09:33] Marcos).

Há ainda a forma de ligar `OrderService.changeStatus` ao módulo de webhooks sem acoplá-los ([09:41] Bruno).

## Decisão

1. **A outbox guarda o payload já renderizado, no momento da inserção** (snapshot) ([09:52] Larissa, [09:52] Diego, [09:52] Bruno: "Decidido").
2. **O filtro de eventos é aplicado na inserção**: se nenhum webhook ativo do customer quer aquele status, nada é inserido, "economiza linha na tabela" ([09:34] Bruno, [09:34] Diego).
3. **A ponte com pedidos é a função `publishWebhookEvent(tx, order, fromStatus, toStatus)`**, que recebe o client transacional (`tx`) da transação de `changeStatus`. O `OrderService` não recebe o repository inteiro do webhook ([09:41] Bruno, [09:41] Diego).
4. O payload é enxuto: `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`; **sem `items`** ([09:43] Diego, [09:44] Bruno). Quem quiser mais detalhe consulta `GET /orders/:id`.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Guardar só `order_id` e renderizar no envio** | Se o pedido mudar depois, o evento refletiria um estado posterior ao da mudança; "tem caso esquisito" ([09:52] Larissa). |
| **Filtrar na hora de enviar** | Insere linhas que nunca serão enviadas; foi a pergunta de Diego ([09:34]) e Bruno optou pela inserção para economizar linhas. |
| **Injetar o repository do webhook no `OrderService`** | Acoplamento maior; "função pura recebendo o tx" basta ([09:41] Diego). |
| **Incluir `items` no payload** | Infla o evento; o cliente já tem `GET /orders/:id` ([09:43] Diego). |

## Consequências

**Positivas**

- O evento é uma foto imutável do estado no momento da mudança, o que dá reprodutibilidade para replay e para o histórico de entregas ([09:34] Marcos pede payload no histórico).
- O worker não precisa consultar `orders` para enviar; menos acoplamento e menos carga.
- A outbox só cresce com eventos que alguém vai receber.

**Negativas / trade-offs**

- A renderização do payload e a consulta aos endpoints ativos acontecem **dentro** da transação de `changeStatus`, que já é pesada; o custo extra é pequeno (sem `items`) mas é custo.
- O payload fica duplicado em `webhook_outbox` e, depois de falha definitiva, em `webhook_dead_letter`.
- Se o contrato do payload evoluir, eventos antigos pendentes saem com o formato antigo.
- Mudanças em um webhook (por exemplo, edição da lista de status) afetam apenas eventos futuros; eventos já inseridos seguem seu curso.

## Referências ao código

- `src/modules/orders/order.service.ts`, `changeStatus`: tem o `tx` (`Prisma.TransactionClient`, alias `TxClient` já declarado no arquivo) e carrega `order` com `items` antes do update; o objeto de pedido é a fonte do snapshot.
- `prisma/schema.prisma`: model `Order` (`orderNumber`, `customerId`, `totalCents`) fornece os campos do payload.
