# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Webhooks de saída para mudança de status de pedido |
| **Status** | Rascunho para implementação (após revisão do [RFC](RFC.md)) |
| **Data** | 2026-10-26 |
| **Documentos** | [PRD](PRD.md) · [RFC](RFC.md) · [Tracker](TRACKER.md) · ADRs [001](adrs/ADR-001-outbox-no-mysql.md) [002](adrs/ADR-002-retry-com-backoff-e-dlq.md) [003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md) [004](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md) [005](adrs/ADR-005-worker-separado-em-polling.md) [006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) [007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) |

**Convenção de leitura.** Itens marcados *(desenho)* são derivações necessárias para implementar e **não** foram ditos literalmente na reunião; vêm sempre ancorados na decisão mais próxima e estão listados no [Tracker](TRACKER.md) com o aviso "(derivado)". O que a reunião fixou aparece sem marca. Arquivos marcados "(novo)" ou listados em "Estrutura nova do módulo" ainda **não existem** e serão criados; todos os demais caminhos de arquivo citados existem no repositório base. Os caminhos de API da reunião (`/webhooks`, `/admin/webhooks/...`) são relativos ao prefixo `/api/v1` que `src/app.ts` já aplica (`app.use('/api/v1', ...)`).

---

## 1. Contexto e motivação técnica

Clientes B2B precisam saber quando o status de um pedido muda, em menos de 10 s, sem fazer polling em `GET /orders` ([09:00] Marcos, [09:02] Marcos). A mudança de status ocorre em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de `this.prisma.$transaction`, que hoje atualiza `orders`, insere em `order_status_history` e debita ou repõe estoque. Chamar o cliente de dentro dessa transação é inviável: cliente lento trava a mudança e cliente fora do ar não pode causar rollback ([09:04] Bruno). Por isso o desenho usa o padrão **outbox** (ADR-001): o evento é gravado atomicamente e entregue por um **worker separado** (ADR-005).

## 2. Objetivos técnicos

| ID | Objetivo | Origem |
|---|---|---|
| FDD-OBJ-01 | Nenhuma mudança de status commitada sem evento (para endpoints inscritos), e nenhum evento sem a mudança que o gerou. | [09:40] Bruno |
| FDD-OBJ-02 | Primeira tentativa de entrega em menos de 10 s após o commit (polling de 2 s). | [09:02] Marcos, [09:09] Diego |
| FDD-OBJ-03 | Falha ou lentidão de um cliente não afeta `PATCH /orders/:id/status` nem a API. | [09:04] Bruno, [09:11] Diego |
| FDD-OBJ-04 | Entrega autenticável (HMAC-SHA256, secret por endpoint) e at-least-once com `X-Event-Id`. | [09:22] Sofia, [09:26] Larissa |
| FDD-OBJ-05 | Reuso integral dos padrões do projeto; nenhuma dependência npm nova. | [09:30] Larissa |

## 3. Escopo e exclusões

**Dentro:** tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_dead_letter` e `webhook_deliveries`; módulo `src/modules/webhooks` (novo); `src/worker.ts` (novo); `publishWebhookEvent`; CRUD de configuração, rotação de secret, histórico de entregas e replay de DLQ.

**Fora (decidido ou adiado na reunião):** e-mail de alerta ao cliente ([09:37] Larissa); rate limiting de saída, a observar ([09:39] Larissa); dashboard visual ([09:40] Larissa); webhooks de entrada, pois o fluxo é só de saída ([09:02] Marcos); arquivamento das linhas entregues após ~30 dias ([09:08] Diego); ordering global e múltiplos workers ([09:13] Larissa); exactly-once ([09:25] Diego); endurecimento de permissões do CRUD ([09:37] Sofia).

**Fora por consequência do código atual (não discutido):** `OrderService.create` cria pedidos em `PENDING` com histórico `fromStatus: null` e **não** emite evento; só `changeStatus` publica, que é o gancho citado em [09:40] Bruno.

## 4. Modelo de dados

Convenções herdadas de `prisma/schema.prisma`: PK `String @id @default(uuid()) @db.Char(36)` ([09:51] Larissa), `@@map` em snake_case para o nome da tabela, campos em camelCase, `@@index`, `onDelete: Cascade` nos filhos. Todas as tabelas abaixo entram em **uma migration nova**, sem alterar colunas existentes. *(desenho: nomes de campos e tabelas auxiliares)*

```prisma
enum WebhookOutboxStatus {
  PENDING      // pendente (inclui os que aguardam retry: nextAttemptAt no futuro)
  PROCESSING   // processando (reservado pelo worker)
  FAILED       // falhou de forma permanente (está na DLQ)
  DELIVERED    // entregue
}

enum WebhookDeliveryResult {
  SUCCESS
  FAILURE
}

model WebhookEndpoint {                         // FDD-DADOS-01
  id                      String   @id @default(uuid()) @db.Char(36)
  customerId              String   @db.Char(36)
  url                     String   @db.VarChar(2048)
  secret                  String   @db.VarChar(128)
  previousSecret          String?  @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json     // lista de OrderStatus que o endpoint quer receber
  active                  Boolean  @default(true)
  createdAt               DateTime @default(now())
  updatedAt               DateTime @updatedAt

  customer Customer        @relation(fields: [customerId], references: [id], onDelete: Cascade)
  outbox   WebhookOutbox[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {                           // FDD-DADOS-02
  id            String              @id @default(uuid()) @db.Char(36)   // é o X-Event-Id
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)                        // sem FK (ver nota)
  eventType     String              @db.VarChar(64)
  payload       Json                // snapshot renderizado na inserção
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)  // requisições HTTP do ciclo atual (o replay zera)
  nextAttemptAt DateTime            @default(now())
  lockedAt      DateTime?
  lastError     String?             @db.VarChar(500)
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook     WebhookEndpoint     @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([status])
  @@index([createdAt])
  @@index([status, nextAttemptAt])
  @@index([webhookId])
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDeadLetter {                       // FDD-DADOS-03
  id            String    @id @default(uuid()) @db.Char(36)
  outboxId      String    @db.Char(36)
  webhookId     String    @db.Char(36)
  eventId       String    @db.Char(36)
  payload       Json
  failureReason String    @db.VarChar(500)   // código WEBHOOK_* + detalhe
  attempts      Int
  failedAt      DateTime  @default(now())
  replayedAt    DateTime?

  outbox WebhookOutbox @relation(fields: [outboxId], references: [id], onDelete: Cascade)

  @@index([webhookId])
  @@index([replayedAt])
  @@map("webhook_dead_letter")
}

model WebhookDelivery {                         // FDD-DADOS-04 (histórico de entregas)
  id           String                @id @default(uuid()) @db.Char(36)
  outboxId     String                @db.Char(36)
  webhookId    String                @db.Char(36)
  attempt      Int                              // do ciclo atual; o replay reinicia a contagem
  startedAt    DateTime                         // início da chamada HTTP
  result       WebhookDeliveryResult
  httpStatus   Int?
  responseBody String?               @db.Text   // truncado em 2 KB
  durationMs   Int
  error        String?               @db.VarChar(500)
  createdAt    DateTime              @default(now())

  outbox WebhookOutbox @relation(fields: [outboxId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}
```

Notas de modelagem:

- Os campos `status` e `createdAt` da outbox têm índice, como pediu Diego ([09:08] Diego). O índice composto `(status, nextAttemptAt)` é *(desenho)* e serve à consulta do worker com backoff.
- `WebhookOutbox.orderId` **não tem FK** para `orders`: `OrderService.delete` permite apagar pedidos `CANCELLED` (`order.service.ts`, método `delete`); uma FK impediria a exclusão por causa dos eventos já gerados. *(desenho)*
- Uma linha de outbox existe **por evento e por endpoint** (um pedido com dois endpoints inscritos gera duas linhas, com `event_id` distintos). O `id` da linha é o `X-Event-Id`. *(desenho)*
- `DELETE` de um webhook apaga em cascata outbox, entregas e DLQ dele, no mesmo padrão `onDelete: Cascade` de `OrderItem` e `OrderStatusHistory`. Eventos pendentes são descartados. *(desenho; ver risco FDD-RISK-07)*
- `tests/setup.ts` limpa tabelas na ordem de dependência; ao implementar, as tabelas novas entram nesse `beforeEach` antes de `customer.deleteMany()` (os testes são código fora do escopo deste pacote documental).

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (FDD-FLX-01)

Ponto de integração: `OrderService.changeStatus`, após `tx.orderStatusHistory.create` e antes do `findUnique` final.

```text
changeStatus(id, input, userId)
└─ prisma.$transaction(tx)
   ├─ order = tx.order.findUnique(... include items)
   ├─ valida from/to (canTransition)            // inalterado
   ├─ debita/repõe estoque                       // inalterado
   ├─ tx.order.update(status = to)               // inalterado
   ├─ tx.orderStatusHistory.create(...)          // inalterado
   ├─ publishWebhookEvent(tx, order, from, to)   // NOVO, mesma transação
   └─ tx.order.findUnique(refreshed)             // inalterado
```

`publishWebhookEvent(tx, order, fromStatus, toStatus)` ([09:41] Bruno, [09:41] Diego) é função que recebe o client transacional (`Prisma.TransactionClient`) e não depende de classes injetadas:

1. Lê `tx.webhookEndpoint.findMany({ where: { customerId: order.customerId, active: true } })`.
2. Mantém só os endpoints cujo `events` contém `toStatus` (filtro **na inserção**, ADR-007; a filtragem é feita em memória, pois a lista por customer é pequena e evita depender de operadores JSON do MySQL). *(desenho)*
3. Se nenhum sobrou, retorna sem inserir ([09:34] Bruno).
4. Para cada endpoint, gera `eventId = uuid v4`, renderiza o **payload completo** (seção 6.9) e insere a linha com `id = eventId` (é o `X-Event-Id`), `status = PENDING`, `attempts = 0`, `nextAttemptAt = now`. Usa `tx.webhookOutbox.createMany`.
5. Qualquer exceção propaga e causa **rollback da mudança de status** ([09:40] Bruno). Registra log `webhook_event_enqueued` (eventId, webhookId, orderId, toStatus) após o `createMany`.

### 5.2 Processamento pelo worker (FDD-FLX-02)

Entry point `src/worker.ts` (novo; `npm run worker`), `PrismaClient` próprio via `createPrismaClient()` ([09:11] Larissa, [09:30] Bruno). Lógica em `src/modules/webhooks/webhook.processor.ts` (novo) ([09:28] Bruno).

A cada **2 s** ([09:09] Diego), um ciclo (`tick`):

1. **Recuperar presos**: linhas `PROCESSING` com `lockedAt` mais antigo que `WEBHOOK_PROCESSING_LOCK_TIMEOUT_MS` voltam para `PENDING`. O valor deve respeitar `WEBHOOK_PROCESSING_LOCK_TIMEOUT_MS >= WEBHOOK_BATCH_SIZE x WEBHOOK_HTTP_TIMEOUT_MS + folga` (padrão 120000), porque o lote é processado em sequência. *(desenho; consequência do at-least-once, ADR-004)*
2. **Reservar lote**: seleciona até `WEBHOOK_BATCH_SIZE` (padrão 10; "batch pequeno", [09:08] Diego) linhas `PENDING` com `nextAttemptAt <= now`, ordenadas por `nextAttemptAt ASC, createdAt ASC` (na primeira tentativa `nextAttemptAt` equivale ao `createdAt`, mantendo a ordem de criação de [09:12] Diego; retentativas e replays entram na fila quando ficam devidos, sem furá-la), e as move para `PROCESSING` com `lockedAt = now` via `updateMany` condicionado a `status = PENDING`.
3. **Processar sequencialmente**, na ordem. Para cada linha:
   1. Carrega o endpoint. Se a linha ou o endpoint foram removidos durante o ciclo (cascata do `DELETE`; a FK falha com `P2003`/`P2025`), registra `webhook_row_vanished` e segue para a próxima linha. O campo `active` **não** é reavaliado aqui: evento já registrado continua sendo entregue.
   2. Mede `Buffer.byteLength(JSON.stringify(payload))`. Se maior que **64 KB** → falha permanente `WEBHOOK_PAYLOAD_TOO_LARGE`, **sem chamada HTTP** ([09:23] Sofia, [09:24] Larissa).
   3. Monta o corpo (string exata que será assinada; endpoint sem secret utilizável → falha permanente `WEBHOOK_SECRET_REQUIRED`), os headers e as assinaturas (seção 6.8).
   4. Faz `POST` com `fetch` global do Node 20 e `AbortSignal.timeout(10_000)` ([09:42] Diego). Redirects não são seguidos (`redirect: 'manual'`). *(desenho)*
   5. Resultado **2xx** = sucesso. Qualquer outra coisa (3xx, 4xx, 5xx, timeout, erro de rede) = falha retentável. *(desenho)*
   6. Insere uma linha em `webhook_deliveries` (`startedAt`, resultado, status HTTP, corpo da resposta truncado em 2 KB, duração).
   7. Sucesso → `status = DELIVERED`, `attempts += 1`, `lockedAt = null`. Falha → fluxo 5.3.
4. Se o lote veio cheio (`WEBHOOK_BATCH_SIZE` linhas), inicia o próximo ciclo na hora, para drenar backlog; senão, dorme até completar 2 s desde o início do ciclo. Ciclos nunca se sobrepõem. *(desenho)*

**Instância única.** Deve existir exatamente um processo `worker` ativo: a ordenação por pedido (ADR-005) e o lock dependem disso. Em deploy, o processo antigo termina antes de o novo subir.

Encerramento: `SIGINT`/`SIGTERM` interrompem o loop, aguardam a entrega em curso, devolvem a `PENDING` as linhas reservadas e ainda não processadas e chamam `prisma.$disconnect()`, como em `src/server.ts`.

### 5.3 Retry com backoff (FDD-FLX-03)

Tabela de espera após cada falha, com `attempts` já incrementado ([09:17] Larissa):

| Falha nº (`attempts` após) | Próxima tentativa em | Observação |
|---|---|---|
| 1ª (1) | +1 min | 1ª retentativa |
| 2ª (2) | +5 min | |
| 3ª (3) | +30 min | |
| 4ª (4) | +2 h | |
| 5ª (5) | +12 h | 5ª retentativa |
| 6ª (6) | — | vai para a DLQ (fluxo 5.4) |

Soma dos intervalos: 14 h 36 min entre a primeira falha e a última tentativa ("quase 15 horas", [09:17] Diego). Isto é **1 tentativa inicial + 5 retentativas = 6 requisições**, a leitura que torna a janela de ~15 h coerente; está em confirmação como RFC-OPEN-06. Constante no código: `WEBHOOK_BACKOFF_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]` (decisão de produto, não variável de ambiente). Em cada falha retentável: `status = PENDING`, `nextAttemptAt = now + backoff[attempts - 1]`, `lastError = <código WEBHOOK_*>`, log `webhook_retry_scheduled`.

### 5.4 DLQ e replay (FDD-FLX-04)

**Ida para a DLQ**, numa única transação do worker: `webhook_outbox.status = FAILED`, `lastError`, e insere em `webhook_dead_letter` (payload, `failureReason`, `attempts`, `failedAt`) ([09:18] Diego). Ocorre ao esgotar as tentativas ou em falha permanente (`WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_SECRET_REQUIRED`), sem consumir backoff. Log `webhook_moved_to_dlq` com `deadLetterId`, que é como o ADMIN obtém o id para o replay (a reunião não pediu endpoint de listagem da DLQ).

**Replay** (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`, role **ADMIN**, [09:18] Diego, [09:36] Sofia), em transação:

1. Busca a linha da DLQ; inexistente → `WEBHOOK_DEAD_LETTER_NOT_FOUND`.
2. Se `replayedAt` já preenchido → `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`. *(desenho)*
3. Recoloca a **mesma linha da outbox** como pendente: `status = PENDING`, `attempts = 0`, `nextAttemptAt = now`, `lockedAt = null`, `lastError = null` ([09:18] Diego: "recoloca na outbox como pendente"). O `event_id` não muda, para o cliente deduplicar (ADR-004). *(desenho)*
4. Marca `replayedAt = now` na linha da DLQ.
5. **Registra em log quem executou**: `webhook_dlq_replayed` com `userId` (de `req.user.id`), `deadLetterId`, `eventId`, `requestId` ([09:36] Sofia).

Limites conhecidos do replay: (a) causa permanente que se repete (`WEBHOOK_PAYLOAD_TOO_LARGE`) falha de novo e gera nova linha na DLQ; (b) o evento reenviado pode chegar depois de eventos mais novos do mesmo pedido (RFC-OPEN-07).

### 5.5 Rotação de secret (FDD-FLX-05)

`POST /api/v1/webhooks/:id/secret/rotate`: gera nova secret com `crypto.randomBytes(32).toString('hex')`, copia a atual para `previousSecret` e grava `previousSecretExpiresAt = now + 24 h` ([09:22] Sofia). Rotação feita durante um grace period ainda ativo substitui a `previousSecret` (a mais antiga deixa de valer; o cliente deve concluir a migração antes de rotacionar de novo, ver FDD-RISK-12). Em cada envio o worker inclui a assinatura da secret anterior **somente se** `previousSecretExpiresAt > now` (avaliado no momento do envio). *(desenho; geração e formato entram na revisão de segurança de Sofia, [09:46])*

## 6. Contratos públicos

Todos os endpoints de configuração usam `authenticate` (JWT Bearer do sistema) e aceitam qualquer role autenticada ([09:36] Marcos, [09:37] Sofia). O `customer_id` **não vem do JWT**: é passado "no body ou no path" ([09:32] Larissa). O contrato abaixo escolhe `customerId` no body (POST) e na query (GET); a query é *(desenho)*.

Erros seguem o envelope existente de `src/middlewares/error.middleware.ts`: `{ "error": { "code": "...", "message": "...", "details": ... } }`.

### 6.1 `POST /api/v1/webhooks` — cadastrar webhook (FDD-CONTRATO-01)

**Requisição** (headers `Authorization: Bearer <jwt>` e `Content-Type: application/json`):

```json
{
  "customerId": "5f0c9d3e-8a54-4c63-9c1e-0b6f3d2a1e77",
  "url": "https://api.atlascomercial.example/hooks/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```

**Resposta** `201 Created`; a `secret` é devolvida **somente aqui** ([09:31] Marcos):

```json
{
  "id": "a3b1c2d4-1111-4222-8333-444455556666",
  "customerId": "5f0c9d3e-8a54-4c63-9c1e-0b6f3d2a1e77",
  "url": "https://api.atlascomercial.example/hooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "9f2c...64 caracteres hexadecimais...e1",
  "createdAt": "2026-11-03T14:00:00.000Z",
  "updatedAt": "2026-11-03T14:00:00.000Z"
}
```

Status: `201`; `400 VALIDATION_ERROR` (schema Zod: URL não `https`, `events` vazio ou com status inexistente); `401 UNAUTHORIZED`; `404 NOT_FOUND` (customer inexistente, `NotFoundError('Customer')`).

Regras: `url` precisa ser `https` ([09:23] Sofia); `events` é lista não vazia de status que `changeStatus` pode produzir: `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`. `PENDING` é recusado, pois nenhuma transição leva a ele (`order.status.ts`) e a criação do pedido não emite evento. *(desenho)*

### 6.2 `GET /api/v1/webhooks?customerId=<uuid>&page=1&pageSize=20` — listar webhooks de um customer (FDD-CONTRATO-02)

**Requisição** (sem corpo):

```http
GET /api/v1/webhooks?customerId=5f0c9d3e-8a54-4c63-9c1e-0b6f3d2a1e77&page=1&pageSize=20
Authorization: Bearer <jwt>
```

**Resposta** `200 OK`, no formato paginado de `src/shared/http/response.ts` (`paginated`). **A secret não é retornada** (*desenho*: [09:22] Diego relata vazamento de secret em log de cliente).

```json
{
  "data": [
    {
      "id": "a3b1c2d4-1111-4222-8333-444455556666",
      "customerId": "5f0c9d3e-8a54-4c63-9c1e-0b6f3d2a1e77",
      "url": "https://api.atlascomercial.example/hooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-03T14:00:00.000Z",
      "updatedAt": "2026-11-03T14:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status: `200`; `400 VALIDATION_ERROR` (`customerId` ausente ou não UUID); `401`.

### 6.3 `PATCH /api/v1/webhooks/:id` — editar (FDD-CONTRATO-03)

**Requisição**: corpo parcial; campos aceitos: `url`, `events`, `active`. A secret não é editável aqui (usa-se a rotação). `active: false` **pausa** o webhook: eventos novos deixam de ser registrados (e não são recuperados ao reativar), enquanto eventos já na outbox continuam sendo entregues. Para manutenção do cliente vale o retry, não a desativação. *(desenho; "estado ativo" é de [09:21] Bruno, sem semântica na reunião)*

```json
{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

**Resposta** `200 OK`, com o objeto atualizado (sem `secret`):

```json
{
  "id": "a3b1c2d4-1111-4222-8333-444455556666",
  "customerId": "5f0c9d3e-8a54-4c63-9c1e-0b6f3d2a1e77",
  "url": "https://api.atlascomercial.example/hooks/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-11-03T14:00:00.000Z",
  "updatedAt": "2026-11-03T16:20:00.000Z"
}
```

Status: `200`; `400 VALIDATION_ERROR`; `401`; `404 WEBHOOK_NOT_FOUND`.

### 6.4 `DELETE /api/v1/webhooks/:id` — remover (FDD-CONTRATO-04)

**Requisição**:

```http
DELETE /api/v1/webhooks/a3b1c2d4-1111-4222-8333-444455556666
Authorization: Bearer <jwt>
```

**Resposta** `204 No Content`, sem corpo (mesma convenção de `DELETE /orders/:id`). Status: `204`; `401`; `404 WEBHOOK_NOT_FOUND`. Remove em cascata eventos pendentes, entregas e DLQ do webhook.

### 6.5 `POST /api/v1/webhooks/:id/secret/rotate` — rotacionar secret (FDD-CONTRATO-05)

**Requisição** (sem corpo):

```http
POST /api/v1/webhooks/a3b1c2d4-1111-4222-8333-444455556666/secret/rotate
Authorization: Bearer <jwt>
```

**Resposta** `200 OK`:

```json
{
  "id": "a3b1c2d4-1111-4222-8333-444455556666",
  "secret": "c71a...nova secret...09",
  "previousSecretExpiresAt": "2026-11-04T14:00:00.000Z"
}
```

Semântica: a secret anterior continua válida até `previousSecretExpiresAt` (24 h) e as entregas incluem as duas assinaturas nesse intervalo ([09:22] Sofia). Status: `200`; `401`; `404 WEBHOOK_NOT_FOUND`. *(desenho: caminho e formato do endpoint; a necessidade do endpoint é de [09:21] Sofia.)*

### 6.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas (FDD-CONTRATO-06)

Devolve as **últimas 100 tentativas** do webhook, da mais recente para a mais antiga, com sucesso/falha, payload, resposta e tempo de resposta ([09:34] Marcos). Sem paginação. *(desenho: 100 por tentativa, não por evento)*

**Requisição** (sem corpo):

```http
GET /api/v1/webhooks/a3b1c2d4-1111-4222-8333-444455556666/deliveries
Authorization: Bearer <jwt>
```

**Resposta** `200 OK`:

```json
{
  "data": [
    {
      "id": "d1e2f3a4-0000-4000-8000-000000000001",
      "eventId": "7c1d6a52-9b0e-4d5f-8f11-2a3b4c5d6e7f",
      "attempt": 1,
      "result": "SUCCESS",
      "httpStatus": 200,
      "responseBody": "{\"ok\":true}",
      "durationMs": 184,
      "error": null,
      "payload": { "event_id": "7c1d6a52-9b0e-4d5f-8f11-2a3b4c5d6e7f", "event_type": "order.status_changed" },
      "createdAt": "2026-11-03T14:05:02.000Z"
    }
  ]
}
```

(`payload` completo conforme a seção 6.9; abreviado aqui.) Status: `200`; `401`; `404 WEBHOOK_NOT_FOUND`.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar DLQ (FDD-CONTRATO-07)

Exige role **ADMIN** ([09:36] Sofia, [09:36] Larissa): `authenticate` + `requireRole('ADMIN')`.

**Requisição** (sem corpo):

```http
POST /api/v1/admin/webhooks/dead-letter/e9f8a7b6-0000-4000-8000-0000000000aa/replay
Authorization: Bearer <jwt de usuário ADMIN>
```

**Resposta** `200 OK`:

```json
{
  "deadLetterId": "e9f8a7b6-0000-4000-8000-0000000000aa",
  "outboxId": "7c1d6a52-9b0e-4d5f-8f11-2a3b4c5d6e7f",
  "eventId": "7c1d6a52-9b0e-4d5f-8f11-2a3b4c5d6e7f",
  "status": "PENDING",
  "replayedAt": "2026-11-03T15:30:00.000Z"
}
```

Status: `200`; `401`; `403 FORBIDDEN` (role não ADMIN); `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`; `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`. Efeito colateral auditado em log (seção 5.4).

### 6.8 Requisição de saída para o cliente (FDD-CONTRATO-08)

`POST <url cadastrada>` com timeout de 10 s ([09:42] Diego).

| Header | Valor | Origem |
|---|---|---|
| `Content-Type` | `application/json` | [09:44] Diego |
| `X-Event-Id` | UUID do evento (id da linha da outbox) | [09:25] Diego |
| `X-Signature` | `sha256=<hex>` do HMAC-SHA256 do corpo; durante o grace period, `sha256=<nova>,sha256=<anterior>` *(desenho do formato)* | [09:20] Sofia, [09:22] Sofia |
| `X-Timestamp` | momento do envio, ISO 8601 *(formato: desenho)*; não faz parte da assinatura | [09:44] Diego |
| `X-Webhook-Id` | id do endpoint cadastrado | [09:44] Sofia, [09:45] Diego |

Semântica esperada do cliente: responder **2xx** para confirmar. Qualquer outro resultado conta como falha e agenda retry. O cliente deve deduplicar por `X-Event-Id` ([09:25] Diego). Exemplo de verificação no cliente (Node):

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

function isValid(rawBody: string, header: string, secret: string): boolean {
  const expected = createHmac('sha256', secret).update(rawBody).digest('hex');
  return header.split(',').some((part) => {
    const got = part.trim().replace(/^sha256=/, '');
    return got.length === expected.length && timingSafeEqual(Buffer.from(got), Buffer.from(expected));
  });
}
```

### 6.9 Payload do evento (FDD-CONTRATO-09)

JSON com `event_id`, `event_type`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`; **sem `items`** ([09:43] Diego). Limite de **64 KB** ([09:24] Larissa).

```json
{
  "event_id": "7c1d6a52-9b0e-4d5f-8f11-2a3b4c5d6e7f",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-03T14:05:00.000Z",
  "order_id": "0b5a7f10-3c2d-4e8a-9d61-7a1c2b3d4e5f",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "5f0c9d3e-8a54-4c63-9c1e-0b6f3d2a1e77",
  "total_cents": 125990
}
```

Os nomes saem de `Order` em `prisma/schema.prisma` (`orderNumber`, `customerId`, `totalCents`) convertidos para snake_case no contrato. Campos adicionais de `Order` podem ser acrescentados sem quebrar clientes; remover ou renomear exige versionar o evento.

## 7. Matriz de erros

Todos estendem `AppError` e usam o prefixo `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa). Os erros de API passam pelo `errorMiddleware` sem mudanças. Os erros "do worker" não saem por HTTP: ficam em `lastError`, `failureReason`, `webhook_deliveries.error` e nos logs; herdam `AppError` por uniformidade e o `statusCode` é nominal.

| ID | Código | HTTP | Quando | Classe base sugerida | Reprocessável |
|---|---|---|---|---|---|
| FDD-ERR-01 | `WEBHOOK_NOT_FOUND` | 404 | Webhook inexistente em PATCH, DELETE, rotate, deliveries | `AppError` direto (`NotFoundError` fixa o código `NOT_FOUND`) | — |
| FDD-ERR-02 | `WEBHOOK_INVALID_URL` | 400 | URL não `https` ou inválida chega ao service ou ao worker (o Zod barra antes com `VALIDATION_ERROR`) | `BadRequestError` (aceita `code`) | — |
| FDD-ERR-03 | `WEBHOOK_SECRET_REQUIRED` | 500 (nominal) | Endpoint sem secret utilizável no momento de assinar; não deve ocorrer | `AppError` | Falha permanente → DLQ |
| FDD-ERR-04 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de id inexistente | `AppError` direto | — |
| FDD-ERR-05 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | Replay de item com `replayedAt` preenchido | `ConflictError` | — |
| FDD-ERR-06 | `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 (nominal) | Payload serializado acima de 64 KB | `UnprocessableEntityError` | Falha permanente → DLQ, sem HTTP |
| FDD-ERR-07 | `WEBHOOK_DELIVERY_TIMEOUT` | 504 (nominal) | Cliente não respondeu em 10 s | `AppError` | Sim, retry com backoff |
| FDD-ERR-08 | `WEBHOOK_DELIVERY_HTTP_ERROR` | 502 (nominal) | Resposta não 2xx (inclui 3xx, porque redirect não é seguido) | `AppError` | Sim |
| FDD-ERR-09 | `WEBHOOK_DELIVERY_NETWORK_ERROR` | 502 (nominal) | DNS, conexão recusada, TLS inválido | `AppError` | Sim |

Erros genéricos já existentes que a feature reutiliza: `VALIDATION_ERROR` 400 (Zod, `validate.middleware.ts`), `UNAUTHORIZED` 401, `FORBIDDEN` 403 (role insuficiente no replay) e `NOT_FOUND` 404 (customer inexistente no POST). Observação sobre FDD-ERR-02: `validate.middleware.ts` converte todo `ZodError` em `VALIDATION_ERROR`; por isso o cliente que cadastra `http://` recebe `400 VALIDATION_ERROR` com `details[0].path = "url"`, e `WEBHOOK_INVALID_URL` cobre a revalidação defensiva no service e no worker. Novas classes entram em `src/shared/errors/http-errors.ts` e são reexportadas por `src/shared/errors/index.ts`, o mesmo lugar de `InsufficientStockError`.

## 8. Estratégias de resiliência

| ID | Mecanismo | Definição | Origem |
|---|---|---|---|
| FDD-RES-01 | Timeout | 10 s por chamada, `AbortSignal.timeout`; estouro = falha retentável | [09:42] Diego |
| FDD-RES-02 | Retries e backoff | 5 retentativas após a inicial: 1 min, 5 min, 30 min, 2 h, 12 h; sem jitter (não discutido) | [09:17] Larissa |
| FDD-RES-03 | DLQ | Tabela `webhook_dead_letter`; replay manual por ADMIN | [09:18] Diego |
| FDD-RES-04 | Fallback | Não há canal alternativo de notificação: e-mail está fora de escopo. O fallback é a DLQ com replay manual | [09:37] Larissa |
| FDD-RES-05 | Atomicidade | Evento e mudança de status na mesma transação; falha da outbox desfaz a mudança | [09:40] Bruno |
| FDD-RES-06 | Recuperação de crash do worker | Linhas `PROCESSING` com lock expirado voltam a `PENDING`; pode gerar duplicata, tolerada por at-least-once | [09:24] Diego |
| FDD-RES-07 | Banco indisponível no worker | Ciclo falha, loga `webhook_worker_tick_failed` e tenta no próximo ciclo; o processo não encerra. Linha removida no meio do ciclo é ignorada com log `webhook_row_vanished` | *(desenho)* |
| FDD-RES-08 | Isolamento | Worker em processo separado: reiniciar a API não afeta entregas | [09:11] Diego |
| FDD-RES-09 | Ordering | Um único worker, sequencial, por `createdAt`; ordenação implícita por `order_id`, não global, e de melhor esforço sob retry | [09:12] Diego, [09:13] Larissa |

## 9. Observabilidade

Sem ferramentas novas: o time pediu reaproveitar o Pino ([09:29] Bruno). Métricas e rastreio são derivados de logs estruturados e de consultas às tabelas.

**Logs** (FDD-OBS-01; Pino, `logger.child({ component: 'webhook-worker' })` no worker; eventos em `snake_case` como `http_request` e `server_started`). *(desenho dos nomes)*

| Evento | Campos principais |
|---|---|
| `webhook_event_enqueued` | `eventId`, `webhookId`, `orderId`, `toStatus` |
| `webhook_delivery_attempt` | `eventId`, `webhookId`, `attempt`, `result`, `httpStatus`, `durationMs` |
| `webhook_retry_scheduled` | `eventId`, `attempt`, `nextAttemptAt`, `errorCode` |
| `webhook_moved_to_dlq` | `eventId`, `deadLetterId`, `outboxId`, `webhookId`, `attempts`, `failureReason` |
| `webhook_dlq_replayed` | `userId`, `deadLetterId`, `eventId`, `requestId` (auditoria, [09:36] Sofia) |
| `webhook_worker_started` / `_stopped` / `_tick_failed` | `intervalMs`, `batchSize`, `err` |

Nunca logar secret, assinatura nem corpo do evento. `redactPaths` em `src/shared/logger/index.ts` ganha `*.secret`, `*.previousSecret` e o header `X-Signature`.

**Métricas** (FDD-OBS-02; calculadas por consultas SQL e/ou agregação dos logs; nomes prontos para o dia em que existir backend de métricas):

| Métrica | Tipo | Uso |
|---|---|---|
| `webhook_first_attempt_latency_ms` | histograma (p50/p95/p99) | SLO (meta de 10 s de [09:02] Marcos; p95 é medição): `webhook_deliveries.startedAt` da primeira entrega do evento menos `outbox.createdAt`, só para eventos sem replay (`replayedAt` nulo na DLQ) e com backlog dentro da capacidade do lote |
| `webhook_outbox_pending_total` e `webhook_outbox_oldest_pending_age_seconds` | gauge | Atraso do worker e crescimento da fila; a idade conta só linhas `PENDING` com `nextAttemptAt <= now`, para não confundir com espera de backoff |
| `webhook_deliveries_total{result}` | contador | Taxa de sucesso por janela |
| `webhook_delivery_duration_ms` | histograma | Clientes lentos (timeout 10 s) |
| `webhook_retries_total` | contador | Saúde dos clientes |
| `webhook_dead_letter_open_total` | gauge (`replayedAt IS NULL`) | Fila de trabalho do operador |
| `webhook_events_per_customer_per_minute` | contador | (derivado) insumo opcional para a futura decisão de rate limiting (RFC-OPEN-01; a reunião só decidiu observar, [09:39] Larissa) |

**Tracing** (FDD-OBS-03): não há OpenTelemetry nem APM no projeto e a reunião não pediu um. A correlação ponta a ponta usa o `eventId` como chave de rastreio: log `webhook_event_enqueued` → `webhook_delivery_attempt` → linha em `webhook_deliveries` → `webhook_moved_to_dlq` → `webhook_dlq_replayed`. O mesmo valor vai ao cliente em `X-Event-Id` e no `event_id` do corpo, e o cliente o correlaciona nos logs dele. Chamadas de API continuam rastreadas por `requestId` (`request-logger.middleware.ts`).

## 10. Integração com o sistema existente

Caminhos abaixo existem no repositório base; para cada um, como o módulo de webhooks se encaixa.

| ID | Arquivo | Integração |
|---|---|---|
| FDD-INT-01 | `src/modules/orders/order.service.ts` | `changeStatus` chama `publishWebhookEvent(tx, order, from, to)` dentro do `$transaction`, depois de `tx.orderStatusHistory.create`. É a **única** alteração de lógica em código existente. O `OrderService` não importa o repository de webhooks. |
| FDD-INT-02 | `src/modules/orders/order.status.ts` | Fonte da máquina de estados (`canTransition`). O webhook não reimplementa transições; só recebe `from` e `to` já validados. Os valores de `events` são o subconjunto do enum `OrderStatus` que `changeStatus` produz (sem `PENDING`). |
| FDD-INT-03 | `src/shared/errors/app-error.ts`, `http-errors.ts`, `index.ts` | Novos erros `WEBHOOK_*` estendem `AppError`, `BadRequestError`, `ConflictError` ou `UnprocessableEntityError` (que aceitam `code`). `NotFoundError` não serve para `WEBHOOK_NOT_FOUND` porque fixa o código `NOT_FOUND`. |
| FDD-INT-04 | `src/middlewares/error.middleware.ts` | Nenhuma alteração: já trata `AppError`, `ZodError` e Prisma. Vale só para a API; o worker trata os próprios erros ([09:29] Bruno). |
| FDD-INT-05 | `src/middlewares/auth.middleware.ts` | `authenticate` nas rotas de configuração; `requireRole('ADMIN')` no replay. O id do usuário vem de `req.user.id` e alimenta o log de auditoria. |
| FDD-INT-06 | `src/middlewares/validate.middleware.ts` e `src/modules/orders/order.schemas.ts` | `validate({ params, body, query })` com schemas Zod em `webhook.schemas.ts`. Modelo: `orderIdParamSchema`, `updateOrderStatusSchema`, `listOrdersQuerySchema` (paginação `page`/`pageSize`). |
| FDD-INT-07 | `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService`, `WebhookController` e os devolve no objeto de controllers. |
| FDD-INT-08 | `src/routes/index.ts` | O tipo `Controllers` ganha `webhooks`; `buildApiRouter` monta `/webhooks` e `/admin/webhooks`. |
| FDD-INT-09 | `src/server.ts` | Modelo de bootstrap e shutdown para o novo `src/worker.ts`. O `server.ts` em si não muda. |
| FDD-INT-10 | `src/config/database.ts` | `createPrismaClient()` é reaproveitada pelo worker, que abre seu **próprio** `PrismaClient` com a mesma `DATABASE_URL` ([09:30] Bruno). O singleton `prisma` exportado não é compartilhado entre processos. |
| FDD-INT-11 | `src/config/env.ts` | Novas variáveis no `envSchema` Zod: `WEBHOOK_POLL_INTERVAL_MS` (2000), `WEBHOOK_BATCH_SIZE` (10), `WEBHOOK_HTTP_TIMEOUT_MS` (10000), `WEBHOOK_PAYLOAD_MAX_BYTES` (65536), `WEBHOOK_PROCESSING_LOCK_TIMEOUT_MS` (120000). |
| FDD-INT-12 | `src/shared/logger/index.ts` | Mesmo `logger` Pino; `redactPaths` recebe os campos de secret. |
| FDD-INT-13 | `src/shared/http/response.ts` | `paginated()` formata a listagem `GET /webhooks`. |
| FDD-INT-14 | `prisma/schema.prisma` | Novos models e enums da seção 4; `Customer` ganha a relação `webhooks WebhookEndpoint[]`. Uma migration aditiva em `prisma/migrations/`. |
| FDD-INT-15 | `package.json` | Script `"worker": "tsx --env-file=.env src/worker.ts"` (arquivo novo) (espelha o `dev`) e, para produção, `node --env-file=.env dist/worker.js` (espelha o `start`). |

Estrutura nova do módulo (mesma forma de `src/modules/orders`): `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, `webhook.publisher.ts` (`publishWebhookEvent`), `webhook.processor.ts` (ciclo do worker), `webhook.signature.ts` (HMAC) e `src/worker.ts`.

## 11. Dependências e compatibilidade

- **Runtime**: Node ≥ 20 (`engines` em `package.json`) fornece `fetch`, `AbortSignal.timeout` e `node:crypto`; **nenhuma dependência npm nova**. `uuid` 11.0.3 já está instalado.
- **Banco**: MySQL 8.0 (`docker-compose.yml`) e Prisma 5.22.0; a migration só adiciona tabelas e enums.
- **Compatibilidade com a API atual**: nenhum contrato existente muda. A resposta de `PATCH /orders/:id/status` continua igual; a latência da transação aumenta levemente por causa de uma leitura e do insert na outbox.
- **Ordem de deploy**: migration, depois API, depois worker. Sem worker, os eventos acumulam na outbox sem afetar a API; ao subir, o worker processa o acúmulo. Só uma instância do worker pode ficar ativa (ver 5.2); sem mecanismo de eleição, isso é responsabilidade do deploy.
- **Rollback**: reverter o código da API e parar o worker; as tabelas ficam sem uso.
- **Dependência externa**: o endpoint do cliente deve falar HTTPS com certificado válido (`WEBHOOK_DELIVERY_NETWORK_ERROR` caso contrário).
- **Dependência de processo**: revisão de segurança de Sofia (mínimo de 2 dias úteis) antes do deploy ([09:46] Sofia); documentação para clientes no portal, a cargo de Marcos ([09:26] Marcos, [09:40] Marcos).

## 12. Critérios de aceite técnicos

| ID | Dado / Quando / Então |
|---|---|
| FDD-AC-01 | Dado um endpoint ativo do customer inscrito em `SHIPPED`, quando `changeStatus` leva o pedido a `SHIPPED`, então existe exatamente uma linha `PENDING` em `webhook_outbox` com payload completo, criada na mesma transação. |
| FDD-AC-02 | Dado que o insert na outbox lança erro, quando `changeStatus` roda, então a mudança de status, o histórico e o estoque sofrem rollback. |
| FDD-AC-03 | Dado que nenhum endpoint do customer ouve o novo status, quando o status muda, então nenhuma linha é inserida na outbox. |
| FDD-AC-04 | Dado um evento pendente, quando o worker roda, então a primeira tentativa ocorre em menos de 10 s (p95) e o envio inclui `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type`. |
| FDD-AC-05 | Dado um corpo e uma secret, quando o cliente recalcula HMAC-SHA256 do corpo, então o valor coincide com `X-Signature`. |
| FDD-AC-06 | Dada uma rotação há menos de 24 h, quando o worker envia, então `X-Signature` traz as duas assinaturas; depois de 24 h, traz só a nova. |
| FDD-AC-07 | Dado um cliente que responde 500, quando o worker tenta, então as retentativas ocorrem em +1 min, +5 min, +30 min, +2 h e +12 h e, na falha seguinte, o evento está em `webhook_dead_letter` e `FAILED`. |
| FDD-AC-08 | Dado um cliente que não responde, quando passam 10 s, então a tentativa é registrada como `WEBHOOK_DELIVERY_TIMEOUT` e reagendada. |
| FDD-AC-09 | Dado um payload acima de 64 KB, quando o worker processa, então nenhuma requisição é feita e o evento vai à DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`. |
| FDD-AC-10 | Dado um item na DLQ, quando um ADMIN chama o replay, então a linha da outbox volta a `PENDING` com o mesmo `event_id` e o log `webhook_dlq_replayed` traz o `userId`; com role OPERATOR, a resposta é `403 FORBIDDEN`. |
| FDD-AC-11 | Dado cadastro com `http://`, quando `POST /webhooks`, então a resposta é `400` e nada é gravado. |
| FDD-AC-12 | Dado `GET /webhooks` e `GET /webhooks/:id/deliveries`, então a `secret` nunca aparece e o histórico traz no máximo 100 itens. |
| FDD-AC-13 | Dado um worker derrubado entre o HTTP e o `UPDATE`, quando reinicia e o lock expira, então o evento é reenviado com o mesmo `X-Event-Id`. |

## 13. Riscos e mitigação

| ID | Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| FDD-RISK-01 | **Cliente lento atrasa a fila**: worker único e sequencial; cada cliente com timeout consome até 10 s e atrasa os eventos seguintes, ameaçando o SLO de 10 s. | Média | Médio | Acompanhar `webhook_outbox_oldest_pending_age_seconds` (idade só de linhas devidas); alerta com limiar e canal não está definido porque o projeto não tem backend de métricas, então vale consulta operacional até lá; o backlog acima de um lote cheio por ciclo também estoura o SLO, por isso o ciclo não dorme com lote cheio; evolução para paralelismo por `order_id` já prevista como "problema do futuro" ([09:13] Diego). |
| FDD-RISK-02 | **Ordering sob retry**: evento antigo em backoff pode ser ultrapassado por um mais novo do mesmo pedido (RFC-OPEN-07). | Média | Médio | Documentar como limitação conhecida; cliente usa `from_status`/`to_status` e `timestamp` para reconciliar. |
| FDD-RISK-03 | **Duplicatas** por crash do worker ou timeout após processamento do cliente. | Alta (esperada) | Baixo | At-least-once + `X-Event-Id` documentado no portal ([09:26] Marcos). |
| FDD-RISK-04 | **Secret guardada em claro** no banco (HMAC exige o valor) e exposta em logs. | Baixa | Alto | Redaction no Pino, secret nunca devolvida em GET, revisão de Sofia antes do deploy ([09:46] Sofia). |
| FDD-RISK-05 | **`X-Timestamp` não assinado**: replay attack não é detectável de forma autenticada (RFC-OPEN-08). | Baixa | Médio | Decisão pendente na revisão de segurança. |
| FDD-RISK-06 | **Destino da URL** (rede interna, localhost, redirects): só o requisito HTTPS foi definido; o worker, que faz chamadas a URLs informadas por clientes, pode ser usado para atingir recursos internos (RFC-OPEN-09). | Baixa | Alto | Redirects não seguidos; restrição de destinos na revisão de segurança. `responseBody` do histórico devolve ao chamador até 2 KB da resposta do destino (a reunião pediu "response" em [09:34] Marcos), o que amplia o impacto. |
| FDD-RISK-07 | **`DELETE` em cascata** apaga histórico e DLQ do endpoint. | Média | Médio | Confirmar com Larissa e Bruno se a remoção deve ser lógica. Há também corrida entre `DELETE` e `changeStatus` (insert na outbox falha com `P2003`, que o `errorMiddleware` não trata, e o `PATCH /orders` responde 500 com rollback); a remoção lógica elimina a corrida. |
| FDD-RISK-08 | **Crescimento da outbox** sem rotina de arquivamento. | Média | Médio | Índices em `status` e `createdAt`; arquivamento como evolução ([09:08] Diego). |
| FDD-RISK-09 | **Sem vínculo usuário→customer**: o JWT e o model `User` não têm `customerId`; qualquer usuário autenticado cadastra, rotaciona secret, apaga e lê entregas de qualquer customer. A reunião aceitou, por ora, que **qualquer role** gerencie ([09:37] Sofia); não tratou do acesso entre customers (RFC-OPEN-10). | Média | Alto | Levar à revisão de Sofia antes do deploy: vínculo usuário→customer com checagem do `customerId`, ou restringir os endpoints a ADMIN até haver o vínculo. |
| FDD-RISK-10 | **Falha de outbox bloqueia mudança de status** (intencional). | Baixa | Alto | Testes e2e da transação (FDD-AC-02); monitorar erros em `changeStatus`. |
| FDD-RISK-11 | **Cadastro duplicado**: nada impede dois webhooks iguais (mesmo `customerId` e `url`), que geram dois eventos com `X-Event-Id` distintos para o mesmo destino. | Baixa | Médio | Avaliar unicidade de (`customerId`, `url`); atenção ao limite de 3072 bytes de índice do MySQL com `url` de 2048 caracteres em utf8mb4. |
| FDD-RISK-12 | **Rotação repetida em menos de 24 h** invalida a secret mais antiga antes do prazo, e o `POST .../secret/rotate` não é idempotente (resposta perdida e nova chamada derrubam a secret que o cliente ainda usa). | Baixa | Médio | Documentar no portal; alternativa: manter as duas anteriores até expirarem. |
