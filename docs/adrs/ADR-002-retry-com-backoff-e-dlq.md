# ADR-002: Política de retry com backoff exponencial e DLQ em tabela separada

- **Status:** Aceita (decidida na reunião técnica; ver `TRANSCRICAO.md` [09:17] Larissa, [09:19] Larissa)
- **Data do registro:** 2026-10-26
- **Decisores:** Larissa, Diego, Bruno, Marcos (aceitou a janela), Sofia (exigiu role ADMIN e auditoria no replay)
- **Documentos relacionados:** [RFC](../RFC.md), [FDD](../FDD.md) (seções "Fluxos detalhados" e "Estratégias de resiliência"), [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar fora do ar ou lento. O worker precisa decidir quando tentar de novo e quando desistir, sem deixar eventos "pendurados para sempre" ([09:15] Diego). Já houve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego).

O timeout de cada chamada HTTP é de 10 segundos; estourar o timeout conta como falha e agenda retry ([09:42] Diego).

## Decisão

1. **Backoff exponencial com teto de tentativas.** Intervalos de espera entre tentativas: **1 min, 5 min, 30 min, 2 h, 12 h** ([09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h"). A soma dos intervalos é 14 h 36 min, que Diego chamou de "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego).
2. **Esgotadas as tentativas, a falha é permanente** e o evento vai para a **DLQ**.
3. **A DLQ é a tabela separada `webhook_dead_letter`**, guardando payload, motivo da falha e timestamp. Serve de evidência para debug e reprocessamento ([09:18] Diego).
4. **Reprocessamento manual** pelo endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego).
5. O replay exige **role ADMIN**, reaproveitando `requireRole`, e **registra em log quem executou**, para auditoria ([09:36] Sofia, [09:36] Larissa).

### Contagem das tentativas (ponto de atenção)

A transcrição diz "5 tentativas" e lista **cinco intervalos**. Os dois só são coerentes com o "quase 15 horas" se os cinco intervalos ocorrem **depois da primeira tentativa de envio**: 1 tentativa inicial + 5 retentativas = 6 requisições HTTP no total. Se "5 tentativas" incluísse a inicial, haveria apenas quatro intervalos (soma de 2 h 36 min), o que contradiz a janela de ~15 h citada.

Reforço: ao rejeitar 3 tentativas, Diego diz que "retentaria três vezes em 30 minutos" ([09:16] Diego), isto é, 1m + 5m + 30m são retentativas depois da tentativa inicial. Esta ADR adota a leitura "1 inicial + 5 retentativas" e a registra como questão em aberto no RFC, para confirmação de Diego.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | O evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| **3 tentativas** ("mais agressivo", [09:16] Bruno) | Janela curta demais: uma indisponibilidade de manhã esgotaria as tentativas em ~30 minutos, e já houve cliente com 2 h de manutenção ([09:16] Diego). |
| **Marcar como `FAILED` na própria outbox, sem tabela de DLQ** | Foi a pergunta de Larissa ([09:17]); Diego preferiu tabela separada por deixar a leitura da outbox principal mais limpa e preservar evidência ([09:18] Diego). |
| **Reprocesso automático da DLQ** (plausível, não discutida) | Reintroduziria o loop de retry sem fim que o teto de tentativas quer evitar; a reunião escolheu replay manual ([09:18] Diego). |

## Consequências

**Positivas**

- Janela de recuperação de ~14 h 36 min cobre manutenções planejadas e quedas longas; Marcos aceitou o limite: "se um cliente meu cair por 15 horas, ele já tá com problema sério" ([09:17] Marcos).
- A DLQ concentra o que precisa de atenção humana, separada da fila de trabalho do worker (a outbox mantém também as linhas `FAILED` e `DELIVERED`).
- Replay é auditável (usuário registrado em log) e restrito a ADMIN.

**Negativas / trade-offs**

- Reprocessar a DLQ é trabalho manual de operação. Não há aviso proativo ao cliente: e-mail de falha ficou para próxima fase ([09:37] Larissa).
- Um evento em retry pode ser ultrapassado por evento posterior do mesmo pedido (ver ADR-005, limitação de ordering).
- O replay reenvia o mesmo `event_id` (ADR-004), então o cliente que já deduplica não processa duas vezes.
- Mover para a DLQ e marcar a outbox como `FAILED` devem ocorrer na mesma transação, ou o evento pode ficar sem registro de destino.

## Referências ao código

- `src/middlewares/auth.middleware.ts`: `requireRole('ADMIN')` (hoje usado apenas em `src/modules/users/user.routes.ts`).
- `src/shared/errors/http-errors.ts`: erros específicos do domínio estendem `AppError` com código próprio (ver ADR-006).
