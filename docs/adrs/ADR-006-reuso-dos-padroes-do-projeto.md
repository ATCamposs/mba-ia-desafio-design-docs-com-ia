# ADR-006: Reuso máximo dos padrões existentes do projeto

- **Data do registro:** 2026-10-26
- **Decisores:** Larissa, Bruno, Diego
- **Documentos relacionados:** [RFC](../RFC.md), [FDD](../FDD.md) (seção "Integração com o sistema existente"), [ADR-005](ADR-005-worker-separado-em-polling.md), [ADR-007](ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)

## Status

Aceita (decidida na reunião técnica; ver `TRANSCRICAO.md` [09:30] Larissa)

## Contexto

A codebase tem convenções claras: cada domínio é um módulo em `src/modules/<dominio>/` com `controller`, `service`, `repository`, `routes` e `schemas` ([09:27] Bruno). Erros usam `AppError` e subclasses com `errorCode` próprio; logging é Pino em todo o projeto; um middleware centralizado trata `AppError`, `ZodError` e erros conhecidos do Prisma ([09:28] Bruno, [09:29] Bruno).

A feature de webhooks é nova e tem um segundo processo (worker), o que cria a tentação de usar ferramentas e convenções próprias.

## Decisão

Reuso máximo do que existe ([09:30] Larissa). Em concreto:

1. **Módulo `src/modules/webhooks` (novo)** com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts` e `webhook.schemas.ts`, igual aos outros módulos ([09:27] Bruno). O processamento do worker fica em arquivo dentro do módulo ([09:28] Bruno).
2. **Erros**: `AppError` e subclasses, com **prefixo `WEBHOOK_`** em todos os códigos do módulo, como `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno, [09:29] Larissa).
3. **Logger**: Pino existente (`src/shared/logger/index.ts`); nenhuma biblioteca de log nova ([09:29] Bruno).
4. **Middleware de erro**: o `errorMiddleware` atual já trata `AppError`, Zod e Prisma e captura os novos erros sem alteração ([09:29] Bruno).
5. **Validação**: schemas Zod via `validate()` de `src/middlewares/validate.middleware.ts` ([09:30] Larissa).
6. **Autorização**: `authenticate` nas rotas de configuração, `requireRole('ADMIN')` no replay ([09:36] Larissa).
7. **Identificadores**: UUID em todas as tabelas novas ([09:51] Larissa).
8. **Prisma**: mesmo banco e mesma `DATABASE_URL`; o worker instancia o próprio `PrismaClient` ([09:30] Bruno; ver ADR-005).
9. **Integração com pedidos**: segue o desenho do ADR-007 (função que recebe o `tx` da transação de `changeStatus`).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| ADR-006-ALT-01 · **Serviço/microsserviço separado de webhooks, com código próprio** (plausível) | Seria mais uma base de código e de deploy para um time pequeno; a decisão da reunião foi "módulo igual aos outros" ([09:30] Larissa). |
| ADR-006-ALT-02 · **Biblioteca de terceiros de fila/webhooks** (plausível) | A reunião descartou infraestrutura nova ([09:07] Diego) e pediu "não vamos botar nada novo" no logging ([09:29] Bruno). |
| ADR-006-ALT-03 · **Códigos de erro livres, sem prefixo** | Quebraria a convenção de código por domínio (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) e dificultaria filtrar erros do módulo em logs e no cliente ([09:29] Larissa). |

## Consequências

**Positivas**

- Menor curva de aprendizado: quem conhece `orders` ou `customers` navega em `webhooks` sem mapa novo.
- O `errorMiddleware` não precisa mudar; o contrato de erro (`{ error: { code, message, details } }`) permanece.
- Revisão e testes seguem a infraestrutura atual (`tests/helpers/factories.ts`, `supertest`).

**Negativas / trade-offs**

- A convenção do projeto é uma camada por classe com injeção manual em `src/app.ts` (`buildControllers`); o módulo herda essa verbosidade.
- `NotFoundError` fixa o código `NOT_FOUND` (`src/shared/errors/http-errors.ts`, linhas 27–31) e não aceita código customizado; `WEBHOOK_NOT_FOUND` exige uma subclasse direta de `AppError`, ao contrário das outras que estendem `ConflictError` ou `UnprocessableEntityError`, que aceitam `code`.
- `validate.middleware.ts` converte qualquer `ZodError` em `VALIDATION_ERROR`; erros de schema (ex.: URL não HTTPS) não saem com código `WEBHOOK_*` a menos que o service valide de novo.
- O worker é um segundo processo e **não passa pelo middleware de erro HTTP**; ele precisa de tratamento de erro e log próprios, ainda com Pino.

## Referências ao código

- `src/app.ts`: `buildControllers` e `buildApp` (registro do novo controller); `src/routes/index.ts`: `buildApiRouter` e o tipo `Controllers` (novo `router.use('/webhooks', ...)`).
- `src/shared/errors/app-error.ts`, `http-errors.ts`, `index.ts`: hierarquia de erros.
- `src/middlewares/error.middleware.ts`, `validate.middleware.ts`, `auth.middleware.ts`.
- `src/modules/orders/*` e `src/modules/customers/*`: modelos de módulo a seguir.
