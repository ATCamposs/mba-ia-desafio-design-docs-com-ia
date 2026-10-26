# Tracker de Rastreabilidade

Mapeia cada item registrado em [PRD](PRD.md), [RFC](RFC.md), [FDD](FDD.md) e [ADRs](adrs/) à origem na transcrição ou no código.

- **Fonte `TRANSCRICAO`**: Localização é `[hh:mm] Falante` de `TRANSCRICAO.md`. Mais de uma referência aparece separada por `;`.
- **Fonte `CODIGO`**: Localização é o caminho de arquivo do repositório base.
- Itens com "(derivado)" são derivações de desenho do FDD: a coluna Localização aponta a fala da reunião que os motiva, não uma decisão literal.
- Os IDs `PRD-*`, `RFC-ALT/OPEN-*`, `FDD-*` e `ADR-00N` aparecem nos próprios documentos, junto ao item. Linhas `*-CODE-*`, `RFC-IMPACT-01` e `RFC-PRAZO-01` registram restrições do código ou da reunião que os documentos citam em texto corrido, sem ID.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Notificar em tempo real: latência < 10 s (meta dita); medição em p95 é escolha do PRD (derivado) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Zero mudanças de status commitadas sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Atender Atlas, MaxDistribuição e Nova Cargo; Atlas quer até fim de novembro (meta "3 de 3" é derivada) | TRANSCRICAO | [09:00] Marcos; [09:45] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Janela de recuperação ≈ 14 h 36 min (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:17] Larissa; [09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Cliente lento ou fora do ar não impacta PATCH /orders/:id/status | TRANSCRICAO | [09:04] Bruno |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo | Aviso por e-mail ao cliente adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo | Dashboard visual descartado (projeto do frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo | Webhooks de entrada fora: fluxo só de saída | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-05 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues (~30 dias) fora da feature | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-06 | docs/PRD.md | Fora de escopo | Ordering global e múltiplos workers: limitação conhecida | TRANSCRICAO | [09:13] Larissa |
| PRD-OUT-07 | docs/PRD.md | Fora de escopo | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-OUT-08 | docs/PRD.md | Fora de escopo | Endurecer permissões do CRUD fica para depois | TRANSCRICAO | [09:37] Sofia |
| PRD-OUT-09 | docs/PRD.md | Fora de escopo | Itens do pedido fora do payload | TRANSCRICAO | [09:43] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com url e lista de status; secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar webhook; ativar/desativar pausa novas notificações (semântica derivada) | TRANSCRICAO | [09:33] Bruno; [09:21] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro por status por webhook, aplicado na inserção | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Registro atômico do evento com a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Envio HTTP POST com payload enxuto e headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:43] Diego; [09:44] Diego; [09:44] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 com a secret do endpoint | TRANSCRICAO | [09:22] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Retry (5 retentativas, 1 min a 12 h) e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay de DLQ por ADMIN com log de quem executou | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret pela API com a anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas (resultado, payload, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | customer_id vem no body ou path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência abaixo de 10 s; polling de 2 s aceito | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once; dedup pelo cliente por X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Envio em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | URL do webhook obrigatoriamente https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64 KB; acima, erro e não envio | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada ao cliente | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint, rotacionável | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordenação implícita por pedido, só com worker único | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Reuso de AppError, Pino, middleware de erro, módulos e códigos WEBHOOK_* | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança com mínimo de 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEC-01 | docs/PRD.md | Decisão | Outbox no MySQL em vez de síncrono ou Redis | TRANSCRICAO | [09:08] Larissa |
| PRD-DEC-02 | docs/PRD.md | Decisão | 5 retentativas com backoff, DLQ separada, replay manual | TRANSCRICAO | [09:17] Larissa |
| PRD-DEC-03 | docs/PRD.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação de 24 h | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-04 | docs/PRD.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-05 | docs/PRD.md | Decisão | Worker separado em polling de 2 s | TRANSCRICAO | [09:10] Larissa; [09:11] Diego |
| PRD-DEC-06 | docs/PRD.md | Decisão | Reuso dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-07 | docs/PRD.md | Decisão | Payload em snapshot e filtro na inserção | TRANSCRICAO | [09:52] Bruno; [09:34] Bruno |
| PRD-RISK-01 | docs/PRD.md | Risco | Risco de perder a Atlas por atraso na entrega | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Risco de cliente não deduplicar eventos duplicados | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Risco de vazamento de secret por cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Risco de endpoint fora do ar por mais de ~15 h (eventos na DLQ) | TRANSCRICAO | [09:17] Marcos |
| PRD-RISK-05 | docs/PRD.md | Risco | Risco de cliente lento atrasar fila (worker único, timeout 10 s) (derivado) | TRANSCRICAO | [09:42] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Risco de sobrecarga do cliente sem rate limiting | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-07 | docs/PRD.md | Risco | Risco de qualquer usuário autenticado gerenciar webhooks de qualquer customer (derivado) | TRANSCRICAO | [09:37] Sofia |
| PRD-AC-01 | docs/PRD.md | Critério de Aceitação | Evento recebido em < 10 s (p95 derivado) após mudança para status inscrito | TRANSCRICAO | [09:02] Marcos; [09:33] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de Aceitação | Webhook inscrito só em DELIVERED não recebe SHIPPED | TRANSCRICAO | [09:33] Marcos |
| PRD-AC-03 | docs/PRD.md | Critério de Aceitação | Falha no registro do evento desfaz a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-AC-04 | docs/PRD.md | Critério de Aceitação | 5 retentativas 1m/5m/30m/2h/12h e depois DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-AC-05 | docs/PRD.md | Critério de Aceitação | Replay só por ADMIN, com log do usuário | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-06 | docs/PRD.md | Critério de Aceitação | URL http:// recusada com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de Aceitação | Secret antiga e nova aceitas por 24 h após a rotação | TRANSCRICAO | [09:22] Sofia |
| PRD-AC-08 | docs/PRD.md | Critério de Aceitação | Evento reenviado mantém o mesmo X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-AC-09 | docs/PRD.md | Critério de Aceitação | Histórico mostra até 100 entregas com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-10 | docs/PRD.md | Critério de Aceitação | Payload acima de 64 KB não é enviado | TRANSCRICAO | [09:24] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | HTTP síncrono em changeStatus descartado: trava transação e força rollback | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams descartado: infra nova, overengineering | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger/notificação do banco descartada: MySQL sem listener nativo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry indefinido descartado: evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | 3 tentativas descartadas: janela de ~30 min não cobre manutenção de 2 h | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global descartada: vazou uma, vaza tudo | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Exactly-once descartado: coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-08 | docs/RFC.md | Alternativa descartada | Marcar FAILED na outbox em vez de DLQ separada: preferida tabela separada | TRANSCRICAO | [09:18] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Questão em aberto: rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Questão em aberto: aviso por e-mail | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Questão em aberto: ordering com vários workers | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Questão em aberto: arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-05 | docs/RFC.md | Questão em aberto | Questão em aberto: endurecer permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-06 | docs/RFC.md | Questão em aberto | Questão em aberto (derivado): contagem 5 tentativas vs 5 intervalos | TRANSCRICAO | [09:17] Diego; [09:17] Larissa; [09:16] Diego |
| RFC-OPEN-07 | docs/RFC.md | Questão em aberto | Questão em aberto (derivado): ordering sob retry | TRANSCRICAO | [09:12] Diego |
| RFC-OPEN-08 | docs/RFC.md | Questão em aberto | Questão em aberto (derivado): X-Timestamp fora da assinatura | TRANSCRICAO | [09:44] Diego |
| RFC-OPEN-09 | docs/RFC.md | Questão em aberto | Questão em aberto (derivado): destinos permitidos da URL; a fala só exige https | TRANSCRICAO | [09:23] Sofia |
| RFC-OPEN-10 | docs/RFC.md | Questão em aberto | Questão em aberto (derivado): autorização por customer; Sofia aceitou qualquer role, não o acesso entre customers | TRANSCRICAO | [09:37] Sofia; [09:32] Marcos |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL na mesma transação de changeStatus | TRANSCRICAO | [09:08] Larissa; [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Decisão | Backoff 1m/5m/30m/2h/12h, DLQ em webhook_dead_letter, replay ADMIN | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace de 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004 | docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id UUID; cliente deduplica | TRANSCRICAO | [09:26] Larissa; [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Worker em processo separado, polling de 2 s, src/worker.ts, single-worker | TRANSCRICAO | [09:10] Larissa; [09:11] Diego; [09:11] Larissa; [09:12] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso de módulo, AppError, Pino, error middleware, prefixo WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Payload em snapshot na inserção, filtro na inserção, publishWebhookEvent(tx,...) | TRANSCRICAO | [09:52] Bruno; [09:41] Bruno; [09:34] Bruno |
| FDD-OBJ-01 | docs/FDD.md | Objetivo | Nenhuma mudança commitada sem evento e nenhum evento sem a mudança | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo | Primeira tentativa em menos de 10 s (polling de 2 s) | TRANSCRICAO | [09:02] Marcos; [09:09] Diego |
| FDD-OBJ-03 | docs/FDD.md | Objetivo | Falha de cliente não afeta PATCH /orders/:id/status | TRANSCRICAO | [09:04] Bruno |
| FDD-OBJ-04 | docs/FDD.md | Objetivo | HMAC-SHA256 por endpoint e at-least-once com X-Event-Id | TRANSCRICAO | [09:22] Sofia; [09:26] Larissa |
| FDD-OBJ-05 | docs/FDD.md | Objetivo | Reuso dos padrões do projeto; sem dependência npm nova (derivado) | TRANSCRICAO | [09:30] Larissa; [09:29] Bruno |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | Tabela webhook_endpoints: url, secret, customer, ativo, eventos (derivado) | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Tabela webhook_outbox: índices em status e created_at, UUID, snapshot; índice composto, sem FK em orderId e uma linha por endpoint são derivados | TRANSCRICAO | [09:08] Diego; [09:51] Larissa; [09:52] Larissa |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | Tabela webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | Tabela webhook_deliveries para histórico das últimas 100 entregas (derivado) | TRANSCRICAO | [09:34] Marcos |
| FDD-FLX-01 | docs/FDD.md | Fluxo | publishWebhookEvent(tx, order, from, to) dentro da transação de changeStatus; filtro em memória é derivado | TRANSCRICAO | [09:41] Bruno; [09:41] Diego |
| FDD-FLX-02 | docs/FDD.md | Fluxo | Worker: polling de 2 s, lote pequeno, ordem por created_at, timeout 10 s, limite 64 KB; ordenação por nextAttemptAt, redirect manual, 2xx único e ciclo sem sleep com lote cheio são derivados | TRANSCRICAO | [09:09] Diego; [09:08] Diego; [09:12] Diego; [09:42] Diego; [09:24] Larissa |
| FDD-FLX-03 | docs/FDD.md | Fluxo | Retry com backoff 1m/5m/30m/2h/12h; leitura "1 inicial + 5 retentativas" é interpretação (RFC-OPEN-06) | TRANSCRICAO | [09:17] Larissa; [09:17] Diego |
| FDD-FLX-04 | docs/FDD.md | Fluxo | DLQ em tabela separada e replay que recoloca na outbox como pendente, com log do usuário; mesma linha com attempts=0 e recusas por replayedAt são derivados | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-FLX-05 | docs/FDD.md | Fluxo | Rotação de secret: anterior válida por 24 h (derivado: caminho e geração) | TRANSCRICAO | [09:22] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks: url, status, secret gerada e devolvida na criação; customerId no body e restrição de events são derivados | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | Endpoint de rotação de secret (derivado: caminho) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries: últimas 100; contagem por tentativa e sem paginação são derivados | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay com role ADMIN | TRANSCRICAO | [09:35] Diego; [09:36] Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Headers de saída Content-Type, X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id; formato de X-Signature e X-Timestamp é derivado | TRANSCRICAO | [09:44] Diego; [09:44] Sofia; [09:45] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Payload JSON enxuto sem items, limite 64 KB | TRANSCRICAO | [09:43] Diego; [09:24] Larissa |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND (404): webhook inexistente | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (400): URL não https | TRANSCRICAO | [09:28] Bruno; [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED: endpoint sem secret utilizável | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND (404) (derivado) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (409) (derivado) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE: payload acima de 64 KB | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT: cliente não respondeu em 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-08 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_HTTP_ERROR: resposta não 2xx (derivado) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-09 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_NETWORK_ERROR: DNS, conexão ou TLS (derivado) | TRANSCRICAO | [09:42] Diego |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | 5 retentativas com backoff; ausência de jitter é desenho (derivado) | TRANSCRICAO | [09:17] Larissa; [09:17] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | DLQ em tabela separada com replay manual | TRANSCRICAO | [09:18] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Sem fallback alternativo: e-mail fora de escopo | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-05 | docs/FDD.md | Resiliência | Atomicidade evento + mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-RES-06 | docs/FDD.md | Resiliência | Recuperação de linhas PROCESSING com lock expirado (derivado) | TRANSCRICAO | [09:24] Diego |
| FDD-RES-07 | docs/FDD.md | Resiliência | Worker sobrevive a falha do banco e tenta no próximo ciclo; linha removida no meio do ciclo é ignorada (derivado) | TRANSCRICAO | [09:11] Diego; [09:09] Diego |
| FDD-RES-08 | docs/FDD.md | Resiliência | Isolamento por processo separado | TRANSCRICAO | [09:11] Diego |
| FDD-RES-09 | docs/FDD.md | Resiliência | Ordering implícito por order_id com worker único | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs Pino estruturados; nunca logar secret; redaction (derivado) | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métricas derivadas de SQL e logs; latência < 10 s (percentil p95 derivado) e insumo de rate limiting | TRANSCRICAO | [09:02] Marcos; [09:39] Larissa |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Sem tracing novo: correlação por eventId e requestId | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent dentro do $transaction | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Máquina de estados reaproveitada; enum OrderStatus nos events | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros WEBHOOK_* estendem AppError/ConflictError/etc.; NotFoundError fixa NOT_FOUND | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Trata os novos AppError sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN'); req.user.id para auditoria | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | validate() com schemas Zod; ZodError vira VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | buildControllers instancia repository, service e controller de webhooks | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Controllers e buildApiRouter montam /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Modelo de bootstrap e shutdown para src/worker.ts | CODIGO | src/server.ts |
| FDD-INT-10 | docs/FDD.md | Integração | createPrismaClient reaproveitada; PrismaClient próprio do worker | CODIGO | src/config/database.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Novas variáveis WEBHOOK_* no schema Zod | CODIGO | src/config/env.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Pino existente; redactPaths ganha campos de secret | CODIGO | src/shared/logger/index.ts |
| FDD-INT-13 | docs/FDD.md | Integração | paginated() na listagem de webhooks | CODIGO | src/shared/http/response.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Novos models; relação em Customer; UUID Char(36) | CODIGO | prisma/schema.prisma |
| FDD-INT-15 | docs/FDD.md | Integração | Novo script worker; engines node >= 20 | CODIGO | package.json |
| FDD-AC-01 | docs/FDD.md | Critério de Aceitação | Mudança para status inscrito gera 1 linha PENDING na mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-02 | docs/FDD.md | Critério de Aceitação | Falha da outbox causa rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-03 | docs/FDD.md | Critério de Aceitação | Sem endpoint inscrito, nada é inserido | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-04 | docs/FDD.md | Critério de Aceitação | Primeira tentativa em < 10 s (p95 derivado) com os 5 headers | TRANSCRICAO | [09:02] Marcos; [09:44] Diego |
| FDD-AC-05 | docs/FDD.md | Critério de Aceitação | Cliente recalcula HMAC-SHA256 do corpo e confere X-Signature | TRANSCRICAO | [09:22] Sofia |
| FDD-AC-06 | docs/FDD.md | Critério de Aceitação | Duas assinaturas por 24 h após rotação; depois só a nova | TRANSCRICAO | [09:22] Sofia |
| FDD-AC-07 | docs/FDD.md | Critério de Aceitação | Retentativas em +1m/+5m/+30m/+2h/+12h e depois DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-AC-08 | docs/FDD.md | Critério de Aceitação | Timeout de 10 s registrado e reagendado | TRANSCRICAO | [09:42] Diego |
| FDD-AC-09 | docs/FDD.md | Critério de Aceitação | Payload > 64 KB não é enviado; destino DLQ é desenho (derivado) | TRANSCRICAO | [09:24] Larissa |
| FDD-AC-10 | docs/FDD.md | Critério de Aceitação | Replay por ADMIN mantém event_id e loga userId; OPERATOR recebe 403 | TRANSCRICAO | [09:36] Sofia; [09:18] Diego |
| FDD-AC-11 | docs/FDD.md | Critério de Aceitação | Cadastro com http:// recusado com 400 | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-12 | docs/FDD.md | Critério de Aceitação | Histórico com no máximo 100 itens; secret nunca em GET é desenho (derivado) | TRANSCRICAO | [09:34] Marcos; [09:31] Marcos |
| FDD-AC-13 | docs/FDD.md | Critério de Aceitação | Worker derrubado entre HTTP e UPDATE reenvia com o mesmo X-Event-Id | TRANSCRICAO | [09:24] Diego; [09:25] Diego |
| FDD-RISK-01 | docs/FDD.md | Risco | Cliente lento atrasa a fila (worker único, timeout 10 s) | TRANSCRICAO | [09:42] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Ordering sob retry | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Duplicatas por crash ou timeout | TRANSCRICAO | [09:25] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Vazamento de secret; guarda em claro no banco e exposição em logs nossos são derivados | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-05 | docs/FDD.md | Risco | X-Timestamp não assinado | TRANSCRICAO | [09:44] Diego |
| FDD-RISK-06 | docs/FDD.md | Risco | Destino da URL (rede interna, redirects) e leitura de resposta pelo histórico (derivado) | TRANSCRICAO | [09:23] Sofia; [09:34] Marcos |
| FDD-RISK-07 | docs/FDD.md | Risco | DELETE em cascata apaga histórico e corrida com changeStatus (padrão onDelete Cascade do schema) (derivado) | CODIGO | prisma/schema.prisma |
| FDD-RISK-08 | docs/FDD.md | Risco | Crescimento da outbox sem arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-09 | docs/FDD.md | Risco | Sem vínculo usuário→customer: qualquer usuário autenticado gerencia webhooks de qualquer customer; a reunião aceitou só qualquer role (derivado) | TRANSCRICAO | [09:37] Sofia; [09:32] Marcos |
| FDD-RISK-10 | docs/FDD.md | Risco | Falha de outbox bloqueia mudança de status (intencional) | TRANSCRICAO | [09:40] Bruno |
| FDD-RISK-11 | docs/FDD.md | Risco | Cadastro duplicado de webhook (mesmo customer e url) gera eventos distintos (derivado) | CODIGO | prisma/schema.prisma |
| FDD-RISK-12 | docs/FDD.md | Risco | Rotação repetida em menos de 24 h invalida a secret mais antiga; endpoint não idempotente (derivado) | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia |
| FDD-CODE-01 | docs/FDD.md | Restrição | OrderService.create não emite evento: só changeStatus publica | CODIGO | src/modules/orders/order.service.ts |
| FDD-CODE-02 | docs/FDD.md | Restrição | outbox.orderId sem FK porque OrderService.delete permite apagar pedidos CANCELLED | CODIGO | src/modules/orders/order.service.ts |
| FDD-CODE-03 | docs/FDD.md | Restrição | Caminhos da API ficam sob o prefixo /api/v1 aplicado em buildApp | CODIGO | src/app.ts |
| FDD-CODE-04 | docs/FDD.md | Restrição | Tabelas novas entram na limpeza de tests/setup.ts ao implementar | CODIGO | tests/setup.ts |
| ADR-006-CODE | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | NotFoundError não aceita código customizado; WEBHOOK_NOT_FOUND estende AppError | CODIGO | src/shared/errors/http-errors.ts |
| ADR-005-CODE | docs/adrs/ADR-005-worker-separado-em-polling.md | Restrição | Bootstrap e shutdown de src/server.ts como modelo do worker | CODIGO | src/server.ts |
| ADR-003-CODE | docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md | Restrição | redactPaths do Pino não cobre *.secret; precisa ser estendido | CODIGO | src/shared/logger/index.ts |
| ADR-001-CODE | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Transação de changeStatus (order, histórico, estoque) onde a outbox entra | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMPACT-01 | docs/RFC.md | Impacto | Alteração crítica em código existente é no changeStatus; o resto é módulo novo, registro em app/rotas, env, logger e script | TRANSCRICAO | [09:40] Bruno; [09:30] Larissa |
| RFC-PRAZO-01 | docs/RFC.md | Restrição | Prazo de três sprints incluindo 2 dias úteis de revisão de segurança | TRANSCRICAO | [09:46] Larissa; [09:46] Sofia |

**Totais:** 180 linhas; 154 com Fonte = TRANSCRICAO (85%); 26 com Fonte = CODIGO. Linhas que carregam alguma derivação de desenho ("derivado"): 41; nelas a Localização aponta a fala que motiva o item, não uma decisão literal.
