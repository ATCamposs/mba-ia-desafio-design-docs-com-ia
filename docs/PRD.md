# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Produto** | Order Management System (OMS) |
| **Responsável de produto** | Marcos (Product Manager) |
| **Status** | Em revisão: consolida RFC, FDD e ADRs; as questões abertas estão no RFC §5 (RFC-OPEN-06 a 10) |
| **Data** | 2026-10-26 |
| **Documentos** | [RFC](RFC.md) · [FDD](FDD.md) · [Tracker](TRACKER.md) · [ADRs](adrs/) |

> Este PRD responde "por que e o quê". A solução técnica está no [RFC](RFC.md), as decisões em [adrs/](adrs/) e a implementação no [FDD](FDD.md); não são repetidas aqui.

## 1. Resumo e contexto da feature

O OMS passa a **notificar clientes B2B por webhook** sempre que o status de um pedido deles mudar. A feature é só de saída: a plataforma envia, o cliente recebe ([09:02] Marcos). Hoje a plataforma não tem nenhum mecanismo de notificação externa; os clientes descobrem mudanças consultando `GET /orders` de tempos em tempos.

## 2. Problema e motivação

- Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) fizeram pedido formal de notificação em tempo real ([09:00] Marcos).
- O polling atual deixa a integração deles lenta e cara ([09:00] Marcos).
- A Atlas indicou que pode migrar para um concorrente se a entrega não ocorrer até o fim do trimestre ([09:00] Marcos); o prazo informado depois foi fim de novembro ([09:45] Marcos).
- "Tempo real" para eles é abaixo de 10 segundos; o que importa é não terem que ficar atualizando manualmente ([09:02] Marcos).

## 3. Público-alvo e cenários de uso

| Persona | Quem é | Cenário |
|---|---|---|
| **Sistema do cliente B2B** (Atlas, MaxDistribuição, Nova Cargo) | Integrações que consomem a API | Recebe um POST quando o pedido passa para um status que ele escolheu; confere a assinatura e deduplica por `X-Event-Id`. |
| **Usuário que representa o cliente** | Usuário autenticado no OMS por JWT ([09:32] Marcos) | Cadastra, edita, remove e lista webhooks; rotaciona a secret; consulta o histórico de entregas. |
| **Administrador da plataforma** (role ADMIN) | Operação interna | Reprocessa eventos que foram para a DLQ ([09:36] Sofia). |
| **Marcos / time de produto** | PM | Documenta a integração no portal de desenvolvedor ([09:40] Marcos). |

Cenários principais: (1) "só quero saber quando vira SHIPPED e DELIVERED" ([09:33] Marcos); (2) o endpoint do cliente cai para manutenção de 2 h e depois volta, e os eventos chegam ([09:16] Diego); (3) o cliente quer saber "quais foram os últimos 100 webhooks que vocês mandaram pra mim" ([09:34] Marcos); (4) um cliente vaza a secret e a troca sem janela de indisponibilidade ([09:21] Sofia, [09:22] Diego).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| PRD-OBJ-01 | Notificar em tempo real | Latência entre o commit da mudança de status e a primeira tentativa de entrega, medida no percentil 95 (o percentil é escolha de medição deste PRD) | **< 10 s** | [09:02] Marcos |
| PRD-OBJ-02 | Não perder notificação | Mudanças de status commitadas, para endpoints inscritos, sem evento registrado | **0** | [09:40] Bruno |
| PRD-OBJ-03 | Entregar no prazo comercial | Clientes pedintes (Atlas, MaxDistribuição, Nova Cargo) com webhook ativo em produção; a Atlas pede a entrega até o fim de novembro | **3 de 3 clientes ativos, com a Atlas até o fim de novembro** (a meta "3 de 3" é deste PRD) | [09:00] Marcos, [09:45] Marcos |
| PRD-OBJ-04 | Cobrir indisponibilidades longas | Janela entre a primeira falha e a última retentativa | **≈ 14 h 36 min** (1 min, 5 min, 30 min, 2 h, 12 h) | [09:17] Larissa, [09:17] Diego |
| PRD-OBJ-05 | Manter a API de pedidos estável | Impacto de cliente lento ou fora do ar sobre `PATCH /orders/:id/status` | **nenhum** (entrega assíncrona) | [09:04] Bruno |

Insumo opcional (derivado): contar eventos por cliente por minuto ajuda a decidir sobre rate limiting no futuro; a reunião só decidiu observar ([09:39] Larissa).

## 5. Escopo

### 5.1 Incluso

- Cadastro, edição, remoção e listagem de webhooks por customer, com filtro por status.
- Notificação de mudança de status de pedido (`order.status_changed`).
- Assinatura HMAC-SHA256, secret por endpoint, rotação com 24 h de convivência.
- Retry com backoff, DLQ e replay manual por ADMIN.
- Histórico de entregas por webhook (últimos 100).

### 5.2 Fora de escopo (descartado ou adiado na reunião)

| ID | Item | Situação | Origem |
|---|---|---|---|
| PRD-OUT-01 | **Aviso por e-mail** quando o webhook do cliente falha repetidamente | Adiado: "fora de escopo dessa fase", talvez próxima, depois de medir o impacto | [09:37] Larissa, [09:38] Marcos |
| PRD-OUT-02 | **Rate limiting de saída** por cliente | Adiado: "observar e decidir depois" | [09:38] Diego, [09:39] Larissa |
| PRD-OUT-03 | **Dashboard visual** para o cliente ver seus webhooks | Descartado agora: só endpoints; painel é projeto separado do time de frontend | [09:39] Marcos, [09:40] Larissa |
| PRD-OUT-04 | **Webhooks de entrada** (cliente enviando para a plataforma) | Fora: o fluxo é só de saída | [09:02] Marcos |
| PRD-OUT-05 | **Arquivamento** de eventos entregues após ~30 dias | Fora do escopo da feature | [09:08] Diego |
| PRD-OUT-06 | **Ordering global** e múltiplos workers em paralelo | Limitação conhecida; só por pedido e com worker único | [09:13] Larissa |
| PRD-OUT-07 | **Exactly-once** | Descartado: entrega é at-least-once com dedup pelo cliente | [09:25] Diego |
| PRD-OUT-08 | **Endurecer permissões** do CRUD de webhooks | Por ora qualquer role autenticada | [09:37] Sofia |
| PRD-OUT-09 | **Itens do pedido no payload** | Descartado: o cliente consulta `GET /orders/:id` | [09:43] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-FR-01 | O cliente **cadastra** um webhook informando `url` e a **lista de status** que quer receber; a plataforma **gera a secret** e a devolve na criação. | [09:31] Marcos |
| PRD-FR-02 | O cliente **edita** um webhook existente, inclusive ativando ou desativando. Desativar pausa novas notificações, sem acúmulo (derivado: "estado ativo" não tinha semântica na reunião). | [09:33] Bruno, [09:21] Bruno |
| PRD-FR-03 | O cliente **remove** um webhook. | [09:33] Bruno |
| PRD-FR-04 | O cliente **lista** os webhooks de um customer. | [09:33] Bruno |
| PRD-FR-05 | Cada webhook **filtra por status** (ex.: só `SHIPPED` e `DELIVERED`); o filtro é aplicado quando o evento é registrado. | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-06 | Quando o status de um pedido muda, o evento é **registrado de forma atômica** com a mudança: nunca status mudado sem evento. | [09:40] Bruno |
| PRD-FR-07 | O evento é **enviado por HTTP POST** ao endpoint cadastrado, com payload enxuto (sem itens) e com identificação do evento, assinatura e horário de envio em headers (contrato no FDD). | [09:43] Diego, [09:44] Diego, [09:44] Sofia |
| PRD-FR-08 | O envio é **assinado com HMAC-SHA256** com a secret do endpoint. | [09:22] Sofia |
| PRD-FR-09 | Em falha, a plataforma **tenta de novo** com intervalos de 1 min, 5 min, 30 min, 2 h e 12 h (a contagem exata de tentativas está em RFC-OPEN-06) e, esgotadas, move o evento para a **DLQ**. | [09:17] Larissa |
| PRD-FR-10 | Um **ADMIN reprocessa** um item da DLQ por endpoint, e a plataforma registra em log quem fez. | [09:18] Diego, [09:36] Sofia |
| PRD-FR-11 | O cliente **rotaciona a secret** pela API; a anterior segue válida por 24 h. | [09:21] Sofia, [09:22] Sofia |
| PRD-FR-12 | O cliente consulta o **histórico das últimas 100 entregas** (sucesso ou falha, payload, resposta, tempo de resposta). | [09:34] Marcos |
| PRD-FR-13 | O `customer_id` é informado na requisição (body ou path), **não** derivado do JWT. | [09:32] Larissa |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-NFR-01 | **Latência**: notificação em menos de 10 s; polling de 2 s aceito. | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-02 | **Confiabilidade**: garantia at-least-once; o cliente deduplica por `X-Event-Id`. | [09:26] Larissa |
| PRD-NFR-03 | **Isolamento**: o envio roda em processo separado da API; falha de cliente não afeta a API. | [09:11] Diego |
| PRD-NFR-04 | **Segurança de transporte**: URL do webhook obrigatoriamente `https`. | [09:23] Sofia |
| PRD-NFR-05 | **Limite de payload** de 64 KB; acima disso, erro e não envio. | [09:24] Larissa |
| PRD-NFR-06 | **Timeout** de 10 s por chamada ao cliente. | [09:42] Diego |
| PRD-NFR-07 | **Secret única por endpoint**, rotacionável. | [09:21] Sofia |
| PRD-NFR-08 | **Ordenação** implícita por pedido, apenas com worker único; sem garantia global. Sob retry ou replay, um evento antigo pode chegar depois de um mais novo do mesmo pedido (RFC-OPEN-07). | [09:13] Larissa |
| PRD-NFR-09 | **Consistência com o projeto**: reuso dos padrões existentes de módulos, erros, logs e validação (ADR-006). | [09:30] Larissa |
| PRD-NFR-10 | **Revisão de segurança** com no mínimo 2 dias úteis antes do deploy. | [09:46] Sofia |

## 8. Decisões e trade-offs principais

Resumo; o raciocínio completo está nos ADRs.

| ID | Decisão | Trade-off | ADR |
|---|---|---|---|
| PRD-DEC-01 | Outbox no MySQL em vez de HTTP síncrono ou Redis | Consistência e zero infra nova, ao custo de carga no MySQL e crescimento de tabela | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| PRD-DEC-02 | 5 retentativas com backoff, DLQ separada, replay manual | Janela de ~15 h; operação manual em vez de retry infinito ou e-mail | [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md) |
| PRD-DEC-03 | HMAC-SHA256 com secret por endpoint e rotação de 24 h | Cliente precisa verificar assinatura; secret guardada recuperável | [ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md) |
| PRD-DEC-04 | At-least-once com `X-Event-Id` | Cliente precisa deduplicar, em troca de simplicidade | [ADR-004](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md) |
| PRD-DEC-05 | Worker separado em polling de 2 s | Latência mínima de 2 s e limite de escala com worker único | [ADR-005](adrs/ADR-005-worker-separado-em-polling.md) |
| PRD-DEC-06 | Reuso dos padrões do projeto | Herda a verbosidade das camadas existentes | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| PRD-DEC-07 | Payload em snapshot e filtro na inserção | Evento reflete o momento da mudança; payload duplicado em tabelas | [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) |

## 9. Dependências

- **Fluxo de pedidos**: a mudança de status do pedido precisa registrar o evento na mesma transação ([09:40] Bruno).
- **Infraestrutura**: o MySQL atual; um processo adicional para o worker; nenhuma infra nova ([09:07] Diego, [09:11] Diego).
- **Segurança**: revisão de Sofia (HMAC e geração de secret) antes do deploy ([09:46] Sofia, [09:49] Sofia).
- **Comunicação com clientes**: Marcos documenta a integração e o comportamento at-least-once no portal de desenvolvedor ([09:26] Marcos, [09:40] Marcos) e atualiza os clientes ([09:49] Marcos).
- **Prazo**: três sprints, incluindo a revisão de segurança ([09:46] Larissa, [09:47] Larissa).

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação | Origem |
|---|---|---|---|---|---|
| PRD-RISK-01 | Perder a Atlas por atraso na entrega | Média | Alto | Prazo de três sprints alinhado com o cliente; agendar cedo a revisão de segurança | [09:00] Marcos, [09:47] Marcos |
| PRD-RISK-02 | Cliente não deduplica e processa eventos duplicados | Média | Médio | Destacar o at-least-once e o uso de `X-Event-Id` no portal | [09:25] Sofia, [09:26] Marcos |
| PRD-RISK-03 | Secret vazada por um cliente | Média | Alto | Secret por endpoint (vazamento contido) e rotação com 24 h de convivência | [09:21] Sofia, [09:22] Diego |
| PRD-RISK-04 | Endpoint do cliente fora do ar por mais de ~15 h: eventos vão para a DLQ e o cliente perde notificações até o replay | Baixa | Médio | Aceito pelo produto; replay manual por ADMIN | [09:17] Marcos |
| PRD-RISK-05 | Cliente lento atrasa as notificações de outros (worker único, timeout de 10 s) | Média | Médio | Monitorar idade do evento mais antigo; evoluir para paralelismo por pedido | [09:13] Diego, [09:42] Diego |
| PRD-RISK-06 | Volume de eventos sobrecarrega o cliente (sem rate limiting) | Baixa | Médio | Observar volume por cliente e decidir depois | [09:38] Diego, [09:39] Larissa |
| PRD-RISK-07 | Qualquer usuário autenticado pode gerenciar webhooks de qualquer customer: nada liga o usuário ao customer | Média | Alto | Definir vínculo usuário→customer ou restringir a ADMIN na revisão de segurança (RFC-OPEN-10) | [09:37] Sofia |

## 11. Critérios de aceitação

| ID | Critério |
|---|---|
| PRD-AC-01 | **Dado** um webhook ativo inscrito em `SHIPPED`, **quando** um pedido do customer passa para `SHIPPED`, **então** o endpoint recebe o evento em menos de 10 s (medido em p95). |
| PRD-AC-02 | **Dado** um webhook inscrito só em `DELIVERED`, **quando** um pedido passa para `SHIPPED`, **então** nenhum evento é gerado para ele. |
| PRD-AC-03 | **Dado** que o registro do evento falha, **quando** o status do pedido muda, **então** a mudança é desfeita. |
| PRD-AC-04 | **Dado** um endpoint que responde 500, **quando** o evento é enviado, **então** há 5 retentativas (1 min, 5 min, 30 min, 2 h, 12 h) e, depois, o evento vai para a DLQ. |
| PRD-AC-05 | **Dado** um item na DLQ, **quando** um usuário ADMIN pede o replay, **então** o evento volta a ser enviado e o log registra o usuário; **quando** um usuário não ADMIN tenta, **então** é recusado. |
| PRD-AC-06 | **Dado** uma URL `http://`, **quando** o cliente cadastra o webhook, **então** o cadastro é recusado com erro de validação. |
| PRD-AC-07 | **Dado** uma rotação de secret, **quando** passam menos de 24 h, **então** entregas aceitam a secret antiga e a nova; **depois** só a nova. |
| PRD-AC-08 | **Dado** um evento reenviado, **então** o `X-Event-Id` é o mesmo da primeira tentativa. |
| PRD-AC-09 | **Dado** um webhook com entregas, **quando** o cliente consulta o histórico, **então** vê até as 100 últimas com resultado, payload, resposta e tempo. |
| PRD-AC-10 | **Dado** um payload acima de 64 KB, **então** ele não é enviado. |

## 12. Estratégia de testes e validação

- **Testes automatizados** no padrão atual do projeto (Vitest e supertest): integração do CRUD de webhooks, autorização (`403` para não ADMIN no replay), assinatura HMAC, filtro por status, atomicidade da outbox em `changeStatus` e fluxo de retry/DLQ com servidor HTTP falso.
- **Cenários de falha**: cliente lento (timeout de 10 s), cliente com 5xx, payload acima de 64 KB, worker derrubado no meio do envio (duplicata esperada).
- **Validação de produto**: teste ponta a ponta do fluxo completo (mudança de status até o endpoint receptor) antes do deploy; medir a latência contra a meta de 10 s.
- **Segurança**: revisão de Sofia sobre HMAC, geração e rotação de secret, em ao menos 2 dias úteis antes do deploy ([09:46] Sofia).
- **Documentação**: portal de desenvolvedor com exemplo de verificação de assinatura e de deduplicação por `X-Event-Id` ([09:26] Marcos, [09:40] Marcos).
