# ADR-005: Worker em processo separado com polling de 2 segundos

- **Data do registro:** 2026-10-26
- **Decisores:** Diego, Larissa, Bruno, Marcos (aceitou a latência)
- **Documentos relacionados:** [RFC](../RFC.md), [FDD](../FDD.md) (seção "Fluxos detalhados"), [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-retry-com-backoff-e-dlq.md)

## Status

Aceita (decidida na reunião técnica; ver `TRANSCRICAO.md` [09:10] Larissa)

## Contexto

Com a outbox em MySQL (ADR-001), alguém precisa ler as linhas pendentes e fazer as chamadas HTTP. A pergunta é como o leitor é acionado e onde ele roda.

O requisito de latência dos clientes é "qualquer coisa abaixo de 10 segundos" ([09:02] Marcos). A aplicação hoje tem um único entry point, `src/server.ts`, que sobe o Express e abre o `PrismaClient` compartilhado de `src/config/database.ts`.

## Decisão

1. **Polling em loop, a cada 2 segundos**: busca os eventos pendentes mais antigos, processa e marca ([09:09] Diego, [09:10] Larissa). A latência mínima de 2 s no pior caso foi aceita ([09:10] Larissa).
2. **O worker roda em processo separado** da API, "senão se a API reinicia, perde o worker" ([09:11] Diego).
3. **Novo entry point `src/worker.ts`** e script `npm run worker`, no mesmo molde de `src/server.ts` ([09:11] Larissa).
4. A lógica de processamento fica dentro do módulo, em arquivo novo do módulo, `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` (a criar; [09:28] Bruno).
5. **Mesmo banco e mesma `DATABASE_URL`, mas `PrismaClient` próprio**, porque o client é por processo ([09:30] Bruno).
6. **Um único worker** por enquanto. A ordenação é por `created_at` da outbox, o que dá ordem implícita por `order_id`; **não há garantia de ordering global** e a garantia só vale enquanto houver um worker ([09:12] Diego, [09:13] Larissa).
7. Escalar para múltiplos workers no futuro exigiria particionar por `order_id` ou usar lock pessimista; fica como "problema do futuro" ([09:13] Diego).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| ADR-005-ALT-01 · **Trigger ou notificação do banco** (equivalente ao `LISTEN/NOTIFY` do Postgres) | MySQL não tem listener nativo; trigger só executa SQL e não avisa processo externo. Avisar o worker exigiria improviso (escrever em arquivo ou chamar endpoint), "fica esquisito" ([09:09] Diego). |
| ADR-005-ALT-02 · **Worker dentro do processo da API** | Reiniciar a API derrubaria o worker ([09:11] Diego). |
| ADR-005-ALT-03 · **Consumidor de Redis Streams** | Já descartado na ADR-001 pelo custo de infraestrutura ([09:07] Larissa, Diego). |
| ADR-005-ALT-04 · **Intervalo de polling menor que 2 s** (plausível) | Mais carga no MySQL sem ganho para o requisito de 10 s; Marcos aceitou 2 s ([09:10] Marcos). |

## Consequências

**Positivas**

- Simples de operar e de entender; sem dependência nova.
- 2 s de polling deixa folga grande para o requisito de 10 s ([09:09] Diego).
- Reinício da API não interrompe entregas; reinício do worker não interrompe a API.
- Reaproveita stack: mesmo Prisma, mesmas variáveis de ambiente (`src/config/env.ts`), mesmo logger.

**Negativas / trade-offs**

- Mais uma unidade de deploy e supervisão (processo `npm run worker`).
- Latência no pior caso de 2 s antes da primeira tentativa, e carga constante de leitura no MySQL mesmo sem eventos.
- Single-worker é limite de escala e de disponibilidade: se o worker cai, as entregas param até ele voltar. As linhas ficam seguras na outbox.
- **A ordenação por pedido é de melhor esforço**: vale para a primeira tentativa de cada evento. Um evento que falhou e está esperando retry pode ser ultrapassado por um evento posterior do mesmo pedido, porque o backoff (ADR-002) reagenda o evento antigo para o futuro. Isso decorre das decisões acima; a reunião não tratou do caso e ele está listado no RFC como questão em aberto.
- Entrega sequencial com timeout de 10 s ([09:42] Diego) significa que um cliente lento atrasa os eventos seguintes na fila; o risco está na seção de riscos do FDD.

## Referências ao código

- `src/server.ts`: modelo de bootstrap e de shutdown (`SIGINT`/`SIGTERM`, `prisma.$disconnect()`) a replicar no novo `src/worker.ts`.
- `src/config/database.ts`: `createPrismaClient()`, reutilizado para criar o client do processo do worker.
- `src/config/env.ts`: schema Zod das variáveis de ambiente, onde entram os parâmetros do worker.
- `package.json`: seção `scripts`, onde entra `"worker"`.
