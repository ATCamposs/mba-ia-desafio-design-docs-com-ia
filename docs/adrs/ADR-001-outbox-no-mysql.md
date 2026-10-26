# ADR-001: Padrão Outbox no MySQL existente

- **Status:** Aceita (decidida na reunião técnica; ver `TRANSCRICAO.md` [09:08] Larissa)
- **Data do registro:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Documentos relacionados:** [RFC](../RFC.md), [FDD](../FDD.md) (seção "Fluxos detalhados"), [ADR-005](ADR-005-worker-separado-em-polling.md), [ADR-007](ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) querem ser notificados quando o status de um pedido muda, em menos de 10 segundos ([09:00] Marcos, [09:02] Marcos). Hoje eles fazem polling em `GET /orders`.

A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que já executa, dentro de um único `this.prisma.$transaction`, quatro operações: `tx.order.update`, `tx.orderStatusHistory.create`, débito de estoque (`debitStock`) e reposição de estoque (`replenishStock`). Bruno descreveu a transação como "já pesada" ([09:04] Bruno).

A aplicação não tem fila, broker, Redis nem mecanismo de eventos. O único componente de dados é o MySQL 8 do `docker-compose.yml`, acessado via Prisma (`prisma/schema.prisma`, `provider = "mysql"`).

Restrições levantadas na reunião:

- Cliente lento ou fora do ar não pode travar a mudança de status de outros pedidos, nem causar rollback da mudança ([09:04] Bruno).
- Não pode existir caso em que o status mudou e o evento não saiu, nem o inverso ([09:40] Bruno, [09:41] Diego).
- O time é pequeno; infraestrutura nova é custo ([09:07] Diego).

## Decisão

Adotar o **padrão Outbox no MySQL existente**: dentro da mesma transação de `changeStatus` que atualiza `orders` e `order_status_history`, inserir uma linha na tabela `webhook_outbox` com o evento. Um processo separado (ADR-005) lê a tabela e faz as chamadas HTTP.

Pontos fixados:

- A tabela tem índice em `status` (valores `PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`, correspondentes a pendente, processando, falhou, entregue) e em `created_at` ([09:08] Diego).
- O worker lê só os pendentes, em batch pequeno ([09:08] Diego).
- Chaves primárias em UUID, seguindo o padrão do projeto ([09:51] Larissa; `prisma/schema.prisma` usa `@id @default(uuid()) @db.Char(36)` em todos os models).
- Se o insert na outbox falhar, a transação inteira faz rollback ([09:40] Bruno).
- Arquivamento de linhas entregues após ~30 dias fica **fora do escopo** desta feature ([09:08] Diego).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **HTTP síncrono dentro de `changeStatus`** | Cliente lento trava a transação e segura locks de `orders` e `products`; cliente fora do ar forçaria escolher entre rollback da mudança de status ou perda do evento ([09:04] Bruno; "síncrono está fora de questão", [09:06] Diego). |
| **Redis Streams ou fila equivalente** | Exige subir infraestrutura nova. Para um time pequeno, "Redis Cluster pra isso é overengineering" ([09:07] Larissa, [09:07] Diego). |
| **Enviar HTTP depois do commit, sem tabela** (plausível, não discutida) | Se o processo cair entre o commit e o envio, o evento se perde: reintroduz o caso "status mudou e evento não saiu" que a reunião proibiu ([09:40] Bruno). |

## Consequências

**Positivas**

- Atomicidade: commit da mudança de status implica evento registrado; rollback descarta o evento junto ([09:06] Diego).
- Nenhuma infraestrutura nova; reaproveita MySQL, Prisma e migrations existentes.
- Falha ou lentidão do cliente fica isolada no worker e não afeta `PATCH /orders/:id/status`.

**Negativas / trade-offs**

- A transação de `changeStatus` ganha uma leitura (endpoints ativos do cliente) e uma ou mais escritas na outbox. É custo aceito em troca da garantia de consistência ([09:41] Diego: "se ficar fora da transação, perde a garantia toda").
- O MySQL passa a ser também uma fila: vazão limitada pelo polling e pelo crescimento da tabela. Mitigações: índices acima e arquivamento futuro (fora do escopo).
- Linhas `DELIVERED` se acumulam até existir rotina de arquivamento. Isso é dívida registrada, não resolvida aqui.
- Falha na inserção da outbox derruba a mudança de status; é intencional ([09:40] Bruno).

## Referências ao código

- `src/modules/orders/order.service.ts`, método `changeStatus` (transação onde a inserção entra).
- `prisma/schema.prisma` (convenção de UUID `Char(36)`, `@@map` em snake_case, `@@index`).
- `docker-compose.yml` (MySQL 8.0 como único serviço).
