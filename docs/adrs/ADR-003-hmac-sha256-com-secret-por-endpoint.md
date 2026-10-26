# ADR-003: Autenticação das entregas com HMAC-SHA256 e secret por endpoint

- **Status:** Aceita (decidida na reunião técnica; ver `TRANSCRICAO.md` [09:22] Sofia)
- **Data do registro:** 2026-10-26
- **Decisores:** Sofia (Eng. de Segurança), Diego, Larissa, Bruno
- **Documentos relacionados:** [RFC](../RFC.md), [FDD](../FDD.md) (seções "Contratos públicos" e "Matriz de erros"), [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md)

## Contexto

Os eventos carregam dados de pedidos e saem para um endpoint fora da infraestrutura da empresa. O cliente precisa conseguir validar que a requisição veio da plataforma e que o payload não foi adulterado no caminho ([09:19] Sofia).

Já houve cliente que vazou secret em log da própria aplicação ([09:22] Diego), então o desenho assume que vazamentos individuais acontecem.

## Decisão

1. **Assinatura HMAC-SHA256 sobre o corpo do request**, enviada no header `X-Signature` ([09:20] Sofia, [09:22] Sofia).
2. **Uma secret única por endpoint de webhook**, não uma secret global da plataforma ([09:21] Sofia). A secret é **gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos).
3. **Rotação pela API**: o cliente pode pedir uma nova secret; a anterior continua **válida por 24 horas em paralelo** (grace period) e depois deixa de valer ([09:21] Sofia, [09:22] Sofia).
4. A configuração do webhook guarda `url`, `secret`, `customer_id` e estado ativo ([09:21] Bruno, confirmado por Sofia).
5. **HTTPS obrigatório** na URL cadastrada; `http` é recusado com erro de validação. Sofia classificou como validação de schema Zod, não como decisão arquitetural ([09:23] Sofia). Está aqui por contexto, e o detalhe fica no FDD.

### Detalhes de desenho derivados (a validar na revisão de segurança)

Estes pontos decorrem da decisão acima mas **não foram fixados na reunião**. Sofia reservou dois dias úteis para revisar HMAC e geração de secret antes do deploy ([09:46] Sofia), e eles entram nessa revisão:

- Durante o grace period, o worker envia no `X-Signature` as duas assinaturas (nova e anterior), separadas por vírgula. Assim o cliente valida com qualquer uma das secrets, o que realiza o "válida em paralelo" da decisão.
- O material assinado é só o corpo, como decidido. O header `X-Timestamp` ([09:44] Diego) **não** entra na assinatura; portanto a detecção de replay pelo cliente com esse header não é autenticada.
- A secret precisa ser recuperável (o HMAC exige o valor em claro no momento da assinatura). Proteção em repouso não foi discutida.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | "Se vaza uma, vaza tudo" ([09:21] Sofia). Com secret por endpoint, o vazamento de um cliente não compromete os outros. |
| **Secret sem rotação** (plausível) | Cliente que vaza a secret ficaria exposto sem caminho de troca; a reunião exigiu rotação com período de convivência ([09:21] Sofia). |
| **Rotação com corte imediato da secret antiga** (plausível) | Obrigaria o cliente a trocar a secret e o código no mesmo instante; o grace period de 24 h dá tempo de migrar os sistemas dele ([09:21] Sofia). |
| **Apenas TLS, sem assinatura** (plausível) | TLS protege o canal, mas não prova ao cliente que a origem é a plataforma na camada de aplicação, que é o requisito de Sofia ([09:19] Sofia). TLS fica como requisito adicional, não substituto. |

## Consequências

**Positivas**

- Padrão de mercado: "todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).
- Raio de impacto de um vazamento limitado a um endpoint.
- Rotação sem janela de indisponibilidade para o cliente.

**Negativas / trade-offs**

- O cliente passa a ter responsabilidade de verificar a assinatura do lado dele.
- Guardar a secret recuperável aumenta o valor do banco como alvo; a proteção em repouso é ponto da revisão de segurança.
- Logs e respostas nunca devem expor a secret. Isso exige estender a lista de redaction do Pino em `src/shared/logger/index.ts`, que hoje cobre `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, mas não `*.secret`.
- Rotação repetida dentro do grace period invalida a secret mais antiga antes das 24 h; o cliente precisa concluir a migração antes de rotacionar de novo.
- Manter duas assinaturas durante o grace period deixa o header `X-Signature` mais longo e exige que o cliente trate lista de valores.

## Referências ao código

- `src/shared/logger/index.ts`: `redactPaths` (precisa incluir os campos de secret).
- `src/modules/customers/customer.schemas.ts` e `src/modules/orders/order.schemas.ts`: padrão de schemas Zod onde a validação de URL HTTPS será escrita.
