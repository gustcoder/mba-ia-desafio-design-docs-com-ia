# ADR-006: Reuso dos Padrões Arquiteturais Existentes do Projeto para o Módulo de Webhooks

## Status

Aceita

## Contexto

O projeto já tem convenções consolidadas: estrutura de módulo (controller/service/repository/routes/
schemas), hierarquia de erros de aplicação, logging estruturado e middleware de erro centralizado. A
equipe discutiu explicitamente como o novo módulo de webhooks deveria se encaixar nessas convenções em
vez de introduzir uma stack paralela.

## Decisão

O módulo de webhooks segue exatamente a mesma estrutura dos módulos existentes (`src/modules/orders/`,
`src/modules/customers/`, etc.): `src/modules/webhooks/` com `webhook.controller.ts`,
`webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, mais um
`webhook.processor.ts`/`webhook.worker.ts` para a lógica consumida pelo entry-point `src/worker.ts`
(ADR-005).

Erros do módulo estendem `AppError` (`src/shared/errors/app-error.ts`), do mesmo jeito que
`InsufficientStockError` e `InvalidStatusTransitionError` já fazem em
`src/shared/errors/http-errors.ts`, com códigos prefixados `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`,
`WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`). Esses erros são capturados automaticamente pelo
middleware de erro já existente (`src/middlewares/error.middleware.ts`), sem exigir nenhuma alteração
nele. O logger Pino já configurado em `src/shared/logger/index.ts` é reaproveitado sem nova instância. O
endpoint administrativo de replay de DLQ reaproveita `requireRole` (`src/middlewares/auth.middleware.ts`)
para exigir a role `ADMIN`.

## Alternativas Consideradas

1. **Criar uma stack de erros, logging ou estrutura de módulo dedicada para webhooks.** Descartada
   implicitamente pela equipe, que optou explicitamente por não introduzir nada novo em termos de logging
   e tratamento de erro.

## Consequências

**Positivas**
- Consistência de código entre o módulo novo e os já existentes, reduzindo a curva de aprendizado do
  time.
- O middleware de erro, a validação Zod e o logging já cobrem o módulo novo sem qualquer mudança em
  código compartilhado.

**Negativas**
- Qualquer limitação dos padrões atuais (por exemplo, ausência de correlação de `request id` em
  processamento assíncrono do worker, já que `request-logger.middleware.ts` só se aplica a requisições
  HTTP da API) é herdada pelo módulo de webhooks e pelo worker.
