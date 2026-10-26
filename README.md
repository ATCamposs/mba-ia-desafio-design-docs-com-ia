# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos**, produzido a partir de `TRANSCRICAO.md` e do código do OMS (Node.js, TypeScript, Prisma e MySQL). O enunciado original está no [repositório base do desafio](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

A tarefa foi transformar a transcrição de uma reunião técnica de ~55 minutos em documentação acionável: PRD, RFC, FDD, ADRs, um Tracker de rastreabilidade e este README do processo. A decisão técnica já estava tomada na reunião (outbox no MySQL, worker separado, retry com DLQ, HMAC, at-least-once, reuso dos padrões do projeto); faltava registrá-la de forma que o time começasse a implementar.

O ponto central do exercício é separar o que **entra** do que **não entra**: a reunião descartou ou adiou e-mail de alerta, rate limiting, dashboard, arquivamento, ordering global e exactly-once, e nada disso aparece como requisito. Toda informação precisa apontar para uma fala da transcrição ou para um arquivo do código; onde o FDD precisou derivar um detalhe de desenho que ninguém disse, ele está marcado e o Tracker diz isso. A entrega é só documental: `src/`, `prisma/` e `tests/` não foram alterados.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** (app desktop do Claude, modelo Claude Sonnet 5.5) | Ferramenta principal: leu a transcrição e o código, escreveu os ADRs, o RFC, o FDD, o PRD, o Tracker e este README, e rodou as validações. |
| **Subagente `claim-verifier`** (Claude Code, 3 execuções) | Verificação independente e somente leitura: 17 afirmações sobre o código e cada linha do Tracker contra a transcrição (as 90 primeiras e as 90 últimas, em duas execuções). |
| **Subagente `adversarial-critic`** (Claude Code) | Crítica adversarial do pacote inteiro, com lentes de contradição, invenção, redundância e furos de implementação. |
| **Script de validação** (Python, local, não versionado) | Checagem mecânica: timestamps e falantes do Tracker existem na transcrição, caminhos de arquivo existem, IDs citados nos documentos têm linha no Tracker, ADRs têm as seções obrigatórias. |

## Workflow adotado

1. **Fork e setup.** Fork do repositório base e clone local.
2. **Contextualização.** Leitura integral da transcrição e dos arquivos que a reunião cita: `order.service.ts`, `order.status.ts`, erros e middlewares, `schema.prisma`, `app.ts`, `server.ts`, rotas, logger, `env.ts` e os testes.
3. **ADRs primeiro** (sete): outbox, retry e DLQ, HMAC com secret por endpoint, at-least-once com `X-Event-Id`, worker separado em polling, reuso dos padrões e payload em snapshot com filtro na inserção.
4. **RFC** sobre os ADRs: proposta concisa, oito alternativas descartadas, questões em aberto separadas entre as da reunião e as que apareceram na elaboração.
5. **FDD** sobre RFC e ADRs: modelo de dados, fluxos, contratos, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção "Integração com o sistema existente" (15 pontos de integração em arquivos reais).
6. **PRD** por último entre os grandes documentos, como consolidação, sem repetir a solução técnica.
7. **Tracker** montado ao final varrendo os documentos prontos, uma linha por item com ID.
8. **Revisão** em ciclos (seção "Iterações e ajustes"), com verificadores independentes e rodadas de correção.
9. **README** por último.

A interação com a IA seguiu uma regra: nada sobe como fato sem conferência. O relatório de um verificador é insumo; cada correção foi conferida contra a transcrição ou o código antes de entrar.

## Prompts customizados

**1. Verificação linha a linha do Tracker** (usado em duas execuções, uma por metade da tabela):

```text
Tarefa read-only (não edite nada). Arquivos: docs/TRACKER.md (ID | Documento | Tipo |
Conteúdo | Fonte | Localização) e TRANSCRICAO.md ([hh:mm] Falante: fala). Código de referência
em uma cópia do repositório base, sem alterações.

Para as linhas 1 a 90 da tabela, verifique: a(s) fala(s) em `Localização` (timestamp +
falante) de fato SUPORTAM o "Conteúdo (resumo)"? Leia a transcrição inteira uma vez.
Classifique cada linha: ok | fraco (a fala toca o assunto mas não sustenta o detalhe
afirmado) | errado (a fala contradiz ou não tem relação). Para linhas marcadas "(derivado)"
aceite 'ok' se a fala motiva o item; diga 'fraco' se o vínculo é só tangencial.
Devolva SOMENTE as linhas não-ok, em tabela: ID | veredito | o que a fala realmente diz |
correção sugerida. No fim, diga quantas linhas estavam ok. Se não conseguir decidir, diga
"nao_provada".
```

**2. Crítica adversarial do pacote** (trecho; o prompt completo trazia as regras do desafio e as quatro lentes):

```text
Ataque o pacote sob estas lentes, com cenário concreto e âncora (arquivo:linha ou
[hh:mm] Falante) para CADA furo; diga também o que atacou e aguentou:
A) Contradições com a transcrição ou o código (números, ordem do backoff, quem decidiu o
   quê, semântica das 5 tentativas, ordering, secret/rotação, headers).
B) Invenção: algo que aparece como requisito/decisão sem origem; ou itens descartados na
   reunião (e-mail, rate limiting, dashboard, arquivamento, ordering global, exactly-once,
   RBAC) virando requisito.
C) Redundância indevida entre RFC e FDD, ou PRD e RFC.
D) Furos de implementação no FDD: um dev conseguiria codar? Verifique o fluxo de replay vs
   `attempts`, o uso de `createMany` com UUIDs, o lock `PROCESSING` e o publish dentro de
   `changeStatus`.
Priorize por severidade. Máximo 15 achados, só os que importam.
```

## Iterações e ajustes

Depois do primeiro rascunho completo foram **três ciclos principais** de revisão e correção.

**Erros que eu mesmo detectei ao escrever**

- **"5 tentativas" não fechava com a conta.** A transcrição diz "total 5 tentativas" e lista cinco intervalos (1 min, 5 min, 30 min, 2 h, 12 h) somando 14 h 36 min, os "quase 15 horas" de Diego. Isso só fecha como 1 envio inicial mais 5 retentativas (6 requisições). Registrei como questão em aberto (RFC-OPEN-06) e, na revisão adversarial, achei a fala que sustenta essa leitura ([09:16] Diego: com 3 tentativas "retentaria três vezes em 30 minutos"), que passei a citar.
- **O código não deixava `WEBHOOK_NOT_FOUND` do jeito que a reunião descreveu.** `NotFoundError` fixa o código `NOT_FOUND` e não aceita um customizado, então o erro precisa estender `AppError` direto. Além disso, `validate.middleware.ts` converte todo `ZodError` em `VALIDATION_ERROR`, então `http://` no cadastro sai como `VALIDATION_ERROR`, e `WEBHOOK_INVALID_URL` ficou como revalidação defensiva. Ajustei FDD e ADR-006.
- **Invenção no PRD.** Um primeiro rascunho do PRD falava em "cliente piloto", que ninguém citou. Removi antes da verificação.

**Ciclo 1: verificação do código e do Tracker (3 execuções do `claim-verifier`).** As 17 afirmações sobre o código foram confirmadas (uma ressalva: `prisma/seed.ts` cria pedidos com status arbitrário, o que não muda o desenho). Das 177 linhas do Tracker, 20 estavam "fracas" (nenhuma errada): a meta "p95" e "3 de 3 clientes ativos" não foram ditas na reunião, localizações incompletas em ADR-005, ADR-007 e PRD-DEC-05, "sem jitter" e "vai à DLQ" eram desenho apresentado como fala. Corrigi localizações e marquei "(derivado)".

**Ciclo 2: crítica adversarial (15 achados, nenhum crítico).** Os que mudaram o conteúdo:

- `active = false` não tinha semântica e, como estava, perdia eventos no cenário de manutenção do cliente. Passou a ser definido como pausa, sem reavaliar eventos já registrados; removi o erro `WEBHOOK_ENDPOINT_INACTIVE` e proibi `PENDING` em `events`.
- O worker dormia mesmo com lote cheio (limite de 5 eventos/s) e o lock de 60 s não cobria um lote de 10 requisições de até 10 s. Agora o ciclo não dorme com lote cheio, o lock tem regra de dimensionamento, o shutdown devolve linhas não processadas e a instância única virou requisito explícito.
- O replay reaproveitava `createdAt` e furava a fila. A ordenação passou a `nextAttemptAt, createdAt` e a latência é medida a partir do início real da chamada.
- Nada ligava usuário a customer. Subi o risco para alto no FDD, no PRD e no RFC (RFC-OPEN-10); a aceitação de Sofia em [09:37] era sobre **role**, não sobre acesso entre customers.
- O PRD se declarava "pronto" enquanto o RFC mantinha questões abertas; virou "em revisão". Reduzi também a duplicação técnica no PRD e no RFC.
- Sete derivações do FDD estavam sem "(derivado)" no Tracker; foram marcadas.

**Ciclo 3: revalidação mecânica.** Rodei de novo o script de validação depois de cada rodada: todos os `[hh:mm] Falante` existem na transcrição, os caminhos de código citados no Tracker existem, todo ID citado nos documentos tem linha no Tracker. Os únicos caminhos "inexistentes" que os documentos mencionam são arquivos novos a criar (`src/worker.ts` e arquivos do módulo `webhooks`), sempre marcados como novos.

Não resolvido de propósito: as questões RFC-OPEN-06 a 10 dependem de Diego, Larissa e Sofia e estão abertas no RFC.

## Como navegar a entrega

Ordem sugerida de leitura (do mais alto nível ao mais detalhado):

1. [`docs/PRD.md`](docs/PRD.md): por que e o quê.
2. [`docs/RFC.md`](docs/RFC.md): proposta técnica, alternativas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/): uma decisão por arquivo.
   - [ADR-001: Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002: Retry com backoff e DLQ](docs/adrs/ADR-002-retry-com-backoff-e-dlq.md)
   - [ADR-003: HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md)
   - [ADR-004: Entrega at-least-once com `X-Event-Id`](docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)
   - [ADR-005: Worker separado em polling](docs/adrs/ADR-005-worker-separado-em-polling.md)
   - [ADR-006: Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
   - [ADR-007: Snapshot do payload e filtro na inserção](docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)
4. [`docs/FDD.md`](docs/FDD.md): como construir, com a seção "Integração com o sistema existente".
5. [`docs/TRACKER.md`](docs/TRACKER.md): de onde veio cada item (transcrição ou código).

`TRANSCRICAO.md` não foi alterado.
