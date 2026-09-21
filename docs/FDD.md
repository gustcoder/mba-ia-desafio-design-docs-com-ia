# FDD: Sistema de Webhooks de Notificação de Pedidos

## Contexto e Motivação Técnica

O OMS precisa notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) sempre que o status de
um pedido muda, substituindo o polling atual em `GET /orders`. A decisão arquitetural
(padrão Outbox + worker dedicado + HMAC + at-least-once) está registrada em `docs/RFC.md` e nos ADRs
correspondentes (`docs/adrs/`). Este documento detalha **como implementar**: modelagem de dados, fluxos
passo a passo, contratos HTTP, matriz de erros, resiliência, observabilidade e os pontos exatos de
integração com o código existente.

O caminho crítico é `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que hoje já
executa, dentro de uma única transação Prisma, a atualização de `order`, a inserção em
`order_status_history` e o ajuste de `stockQuantity`. A feature acrescenta a esse mesmo bloco transacional
a inserção do evento de webhook.

## Objetivos Técnicos

- Emitir um evento por mudança de status de pedido, filtrado pelos status que cada webhook do customer
  assina, com latência de entrega ponta a ponta compatível com o requisito de negócio de "abaixo de 10
  segundos".
- Garantir atomicidade: o evento só existe se a transação de negócio (`changeStatus`) commitou.
- Garantir autenticidade e integridade do payload entregue via HMAC-SHA256 por endpoint.
- Tolerar indisponibilidade temporária do cliente via retry com backoff e DLQ.
- Não introduzir infraestrutura nova (sem broker de filas externo); rodar sobre MySQL/Prisma já existentes.
- Seguir os padrões arquiteturais já estabelecidos no projeto (módulos, `AppError`, Pino, error middleware).

## Escopo e Exclusões

**Em escopo:**
- CRUD de configuração de webhook por customer (URL, secret, lista de status assinados, ativo/inativo).
- Rotação de secret com grace period de 24h.
- Emissão transacional de evento na outbox a partir de `changeStatus`.
- Worker dedicado em polling de 2s, com timeout de 10s por chamada HTTP.
- Retry com backoff exponencial (5 tentativas) e Dead Letter Queue.
- Endpoint administrativo de replay manual de DLQ (role `ADMIN`).
- Histórico de entregas por webhook (últimas entregas, sucesso/falha, payload, resposta, tempo de resposta).
- Assinatura HMAC-SHA256, validação de URL HTTPS, limite de payload de 64KB.

**Fora de escopo / explicitamente adiado** (ver `docs/PRD.md` para o racional de produto):
- Notificação proativa (e-mail) ao cliente em caso de falhas recorrentes — fase futura.
- Rate limiting de envio por cliente — apenas observar, sem implementação nesta fase.
- Painel visual para o cliente acompanhar webhooks — projeto separado do time de frontend.
- Suporte a múltiplos workers em paralelo / particionamento de ordering — limitação conhecida, não
  implementada agora.
- Arquivamento automático de eventos entregues (após ~30 dias) — mencionado, não desta feature.

## Modelagem de Dados (novas tabelas)

Seguindo o padrão de `prisma/schema.prisma` (UUID `@db.Char(36)`, `@@map` em snake_case, índices
explícitos):

- **`WebhookEndpoint`** (`@@map("webhook_endpoints")`): `id`, `customerId`, `url`, `activeSecret`,
  `previousSecret` (nullable), `previousSecretExpiresAt` (nullable), `events` (lista de `OrderStatus`
  assinados — `Json` ou tabela associativa), `active` (boolean), `createdAt`, `updatedAt`. Índice em
  `customerId`.
- **`WebhookOutboxEvent`** (`@@map("webhook_outbox_events")`): `id` (UUID = `event_id`), `webhookEndpointId`,
  `orderId`, `eventType`, `payload` (`Json`, snapshot renderizado), `status`
  (`PENDING | PROCESSING | DELIVERED | FAILED`), `attempts`, `nextAttemptAt`, `createdAt`, `updatedAt`.
  Índices em `status` e `createdAt` (usados pelo polling do worker).
- **`WebhookDeadLetter`** (`@@map("webhook_dead_letters")`): `id`, `webhookOutboxEventId`, `payload`,
  `reason`, `failedAt`. Índice em `webhookOutboxEventId`.
- **`WebhookDelivery`** (`@@map("webhook_deliveries")`): histórico consultável via
  `GET /webhooks/:id/deliveries` — `id`, `webhookEndpointId`, `webhookOutboxEventId`, `success` (boolean),
  `responseStatusCode` (nullable), `responseTimeMs`, `attemptedAt`. Índice em `webhookEndpointId`.

## Fluxos Detalhados

### 1. Criação do evento na outbox (dentro de `changeStatus`)

1. `OrderService.changeStatus` inicia `prisma.$transaction` (já existente).
2. Valida a transição (`canTransition`), debita/repõe estoque, atualiza `order.status` e insere em
   `order_status_history` — inalterado.
3. **Novo passo:** chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`. Essa função:
   a. Busca os `WebhookEndpoint` ativos do `customerId` do pedido cujo `events` contenha `toStatus`.
   b. Se nenhum endpoint assina esse status, retorna sem inserir nada (economiza linha na outbox).
   c. Para cada endpoint correspondente, renderiza o payload (snapshot: `event_id` novo UUID,
      `event_type = "order.status_changed"`, `timestamp` ISO 8601, `order_id`, `order_number`,
      `from_status`, `to_status`, `customer_id`, `total_cents` — sem `items`).
   d. Se o payload renderizado ultrapassar 64KB, não insere e registra o erro `WEBHOOK_PAYLOAD_TOO_LARGE`
      nos logs (não interrompe a transação de negócio — ver Matriz de Erros).
   e. Insere uma linha em `webhook_outbox_events` por endpoint correspondente, com `status = PENDING`.
4. `tx.$transaction` commita. Se qualquer etapa (incluindo a inserção na outbox) falhar, tudo é revertido
   — não há caso de status mudado sem evento correspondente.

### 2. Processamento pelo worker

1. `src/worker.ts` inicia, conecta ao MySQL com seu próprio `PrismaClient`, entra em loop de polling a
   cada 2 segundos.
2. A cada ciclo, busca um lote pequeno (ex.: 20) de eventos com `status = PENDING` e
   `nextAttemptAt <= now()`, ordenados por `createdAt` (ordering por `order_id`, ver ADR-005).
3. Marca os eventos selecionados como `PROCESSING` (claim atômico, evita duplo processamento mesmo em
   cenário de futura escala).
4. Para cada evento: monta os headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`,
   `Content-Type: application/json`), assina o payload com HMAC-SHA256 usando a `activeSecret` do
   endpoint, e faz `POST` para a `url` cadastrada com timeout de 10 segundos.
5. Em caso de sucesso (2xx): grava `WebhookDelivery` (`success = true`), marca o evento como `DELIVERED`.
6. Em caso de falha (timeout, erro de conexão, status HTTP não-2xx): grava `WebhookDelivery`
   (`success = false`, motivo) e segue para o fluxo de retry.

### 3. Retry

1. Ao falhar, incrementa `attempts` e calcula o próximo `nextAttemptAt` pela tabela de backoff:
   tentativa 1 → +1 min, 2 → +5 min, 3 → +30 min, 4 → +2h, 5 → +12h.
2. Evento volta para `status = PENDING` com `nextAttemptAt` no futuro; o worker só o reconsidera quando
   esse horário chegar.
3. Se `attempts` já atingiu 5 e a tentativa atual também falhou, segue para o fluxo de DLQ.

### 4. Dead Letter Queue (DLQ)

1. Ao esgotar as 5 tentativas, o evento é marcado `status = FAILED` e uma linha é inserida em
   `webhook_dead_letters` com o payload, o motivo da última falha e o timestamp.
2. Um operador com role `ADMIN` pode disparar `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
   (ver Contratos Públicos), que recria um `WebhookOutboxEvent` com `status = PENDING` e `attempts = 0`
   a partir do payload salvo na DLQ, e registra em log/auditoria qual usuário fez o replay.

## Contratos Públicos

Todos os endpoints ficam sob `/api/v1`, seguem o padrão de autenticação (`authenticate`) e validação
(`validate({...})`) já usado pelos outros módulos. Erros seguem o formato padrão do error middleware:
`{ "error": { "code": "...", "message": "...", "details": {...} } }`.

### 1. `POST /api/v1/webhooks` — cadastrar webhook

Requisição:
```json
{
  "customerId": "8f14e45f-ceea-467e-a0a0-000000000001",
  "url": "https://api.atlascomercial.com.br/webhooks/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"]
}
```
Resposta `201 Created` (secret retornada apenas nesta chamada):
```json
{
  "id": "b2a1c9e0-1111-4a2b-9c3d-000000000010",
  "customerId": "8f14e45f-ceea-467e-a0a0-000000000001",
  "url": "https://api.atlascomercial.com.br/webhooks/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "secret": "whsec_3f7a9c...",
  "active": true,
  "createdAt": "2026-09-21T13:00:00.000Z"
}
```
Erros possíveis: `400 WEBHOOK_INVALID_URL` (URL não-HTTPS), `400 VALIDATION_ERROR` (customerId inválido,
`events` vazio ou com valor fora do enum `OrderStatus`), `404 NOT_FOUND` (customer inexistente).

### 2. `GET /api/v1/webhooks?customerId=...&page=1&pageSize=20` — listar webhooks

Resposta `200 OK` (mesmo formato de paginação de `src/shared/http/response.ts`):
```json
{
  "data": [
    {
      "id": "b2a1c9e0-1111-4a2b-9c3d-000000000010",
      "customerId": "8f14e45f-ceea-467e-a0a0-000000000001",
      "url": "https://api.atlascomercial.com.br/webhooks/oms",
      "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2026-09-21T13:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
A secret nunca é retornada em listagens ou detalhes — apenas na criação e na rotação.

### 3. `PATCH /api/v1/webhooks/:id` — editar webhook

Requisição:
```json
{ "events": ["SHIPPED", "DELIVERED"], "active": true }
```
Resposta `200 OK`: mesmo formato do item do endpoint 2 (sem secret).
Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`, `400 VALIDATION_ERROR`.

### 4. `DELETE /api/v1/webhooks/:id` — remover webhook

Resposta `204 No Content`. Erros: `404 WEBHOOK_NOT_FOUND`.

### 5. `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Resposta `200 OK`:
```json
{
  "id": "b2a1c9e0-1111-4a2b-9c3d-000000000010",
  "secret": "whsec_9d21ff...",
  "previousSecretValidUntil": "2026-09-22T13:00:00.000Z"
}
```
A secret anterior continua válida por 24h (`previousSecretValidUntil`). Erros: `404 WEBHOOK_NOT_FOUND`.

### 6. `GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=20` — histórico de entregas

Resposta `200 OK`:
```json
{
  "data": [
    {
      "id": "d1e2f3a4-2222-4b3c-9d4e-000000000101",
      "eventId": "6c9c2c2a-3333-4c4d-9e5f-000000000201",
      "success": true,
      "responseStatusCode": 200,
      "responseTimeMs": 184,
      "attemptedAt": "2026-09-21T13:00:02.000Z"
    },
    {
      "id": "d1e2f3a4-2222-4b3c-9d4e-000000000102",
      "eventId": "6c9c2c2a-3333-4c4d-9e5f-000000000200",
      "success": false,
      "responseStatusCode": 503,
      "responseTimeMs": 10000,
      "attemptedAt": "2026-09-21T12:59:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 2, "totalPages": 1 }
}
```
Limitado às últimas 100 entregas por webhook. Erros: `404 WEBHOOK_NOT_FOUND`.

### 7. `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar item da DLQ (role `ADMIN`)

Sem corpo de requisição. Resposta `202 Accepted`:
```json
{ "outboxEventId": "6c9c2c2a-3333-4c4d-9e5f-000000000200", "status": "PENDING" }
```
Erros: `404 WEBHOOK_NOT_FOUND` (item de DLQ inexistente), `401 UNAUTHORIZED` (sem token),
`403 FORBIDDEN` (autenticado mas não é `ADMIN` — reaproveita `requireRole('ADMIN')`),
`409 WEBHOOK_ALREADY_PROCESSED` (item já reprocessado anteriormente).

### Headers de entrega (worker → cliente, não é uma rota da nossa API)

Toda chamada HTTP feita pelo worker para a `url` cadastrada carrega:
`X-Event-Id` (UUID do evento), `X-Signature` (HMAC-SHA256 do corpo), `X-Timestamp` (ISO 8601 do envio),
`X-Webhook-Id` (id do `WebhookEndpoint`), `Content-Type: application/json`.

## Matriz de Erros (`WEBHOOK_*`)

| Código | Status HTTP | Cenário |
|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `webhookId`, item de DLQ ou delivery referenciado não existe |
| `WEBHOOK_INVALID_URL` | 400 | URL cadastrada/atualizada não é HTTPS |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação de assinatura acionada para um webhook sem secret ativa válida (estado inconsistente) |
| `WEBHOOK_INVALID_EVENT_FILTER` | 400 | Campo `events` contém valor fora do enum `OrderStatus` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload renderizado do evento excede 64KB no momento da inserção na outbox |
| `WEBHOOK_INACTIVE` | 409 | Operação (ex.: replay) tentada sobre um webhook desativado |
| `WEBHOOK_ALREADY_PROCESSED` | 409 | Replay solicitado para item de DLQ que já foi reenfileirado |

Todas seguem o padrão de `AppError` (`src/shared/errors/app-error.ts`): `statusCode` + `errorCode` +
`details` opcionais, capturadas automaticamente pelo error middleware existente.

**Motivos de falha de entrega** (não são erros HTTP da nossa API; são valores persistidos em
`webhook_dead_letters.reason` e nos logs do worker): `delivery_timeout` (sem resposta em 10s),
`delivery_connection_error` (falha de DNS/conexão), `delivery_http_error` (resposta 4xx/5xx do cliente),
`max_retries_exceeded` (motivo final ao entrar na DLQ).

## Estratégias de Resiliência

- **Timeout:** 10 segundos por chamada HTTP do worker ao cliente.
- **Retry:** backoff exponencial fixo, 5 tentativas (1m/5m/30m/2h/12h) — ver ADR-002.
- **Fallback:** esgotadas as tentativas, o evento vai para a DLQ; não há fallback automático de canal
  (e-mail fica para fase futura, ver Escopo).
- **Isolamento de falhas:** uma falha de entrega para um `WebhookEndpoint` não afeta o processamento de
  outros eventos/endpoints — o worker processa em lote, e cada item tem seu próprio ciclo de
  retry/backoff.
- **Idempotência do lado do cliente:** garantida por `X-Event-Id` estável entre tentativas do mesmo
  evento (ADR-004); a plataforma não tenta implementar exactly-once.
- **Proteção contra payload anômalo:** o limite de 64KB é validado antes da inserção na outbox, para não
  propagar payloads inconsistentes até a tentativa de entrega.

## Observabilidade

**Métricas** (expostas pelo processo do worker e da API, seguindo o mesmo formato numérico/rotulado já
usado implicitamente pelos logs Pino do projeto):
- `webhook_outbox_pending_count` (gauge) — eventos com `status = PENDING` no momento da coleta.
- `webhook_delivery_attempts_total` (counter, labels `webhook_id`, `result=success|failure`).
- `webhook_delivery_duration_seconds` (histograma da duração da chamada HTTP de entrega).
- `webhook_dead_letter_total` (counter) — eventos que esgotaram as 5 tentativas.
- `webhook_replay_total` (counter, label `actor_user_id`) — replays manuais de DLQ.

**Logs** (Pino, reaproveitando `src/shared/logger/index.ts` sem nova instância — apenas novos eventos
estruturados): `webhook_event_enqueued`, `webhook_delivery_attempt`, `webhook_delivery_success`,
`webhook_delivery_failed`, `webhook_dead_letter_created`, `webhook_dead_letter_replayed`. Cada log inclui
`event_id`, `webhook_id`, `order_id` e, quando aplicável, `attempt` e o motivo de falha. Recomenda-se
adicionar `'*.secret'` à lista `redactPaths` de `src/shared/logger/index.ts` para nunca vazar secrets de
webhook em log — mesma preocupação de segurança que já motivou a exigência de rotação de secret no
cadastro do cliente (ADR-003), agora aplicada também aos nossos próprios logs.

**Tracing:** o projeto não usa tracing distribuído hoje. Para correlação mínima fora do ciclo HTTP da
API (o worker roda fora de `request-logger.middleware.ts`, que só se aplica a requisições Express), cada
ciclo de polling do worker gera um `worker_run_id` interno, e `event_id` é usado como correlation id
primário entre "evento enfileirado" → "tentativa(s) de entrega" → "sucesso ou DLQ" nos logs.

## Dependências e Compatibilidade

- **Banco:** novas tabelas via migration Prisma (`prisma/migrations/`, seguindo o padrão da migration
  `20260519182739_init` já existente); nenhuma dependência de banco além do MySQL já usado.
- **Runtime:** worker roda no mesmo runtime Node `>=20` já exigido pelo projeto (`package.json` →
  `engines.node`), sem framework adicional.
- **Bibliotecas:** HMAC-SHA256 usa o módulo `crypto` nativo do Node (sem nova dependência); geração de
  `event_id`/`webhookId` reaproveita a biblioteca `uuid` já presente nas dependências do projeto.
- **Infraestrutura:** nenhuma infraestrutura nova (sem Redis, sem broker de mensageria) — decisão
  registrada em ADR-001.
- **Compatibilidade:** o novo processo (`src/worker.ts`) precisa da mesma `DATABASE_URL` da API, mas
  instancia seu próprio `PrismaClient` (ADR-005); não compartilha estado em memória com a API.

## Critérios de Aceite Técnicos

- [ ] Evento é inserido em `webhook_outbox_events` somente dentro da mesma transação de `changeStatus`,
      nunca em um caminho separado.
- [ ] Teste de integração comprova que, se a transação de `changeStatus` sofrer rollback, nenhum evento
      correspondente permanece na outbox.
- [ ] Assinatura enviada em `X-Signature` é validável pelo cliente com a `activeSecret` retornada na
      criação do webhook.
- [ ] Após rotação de secret, a secret anterior continua validando assinaturas por exatamente 24h e passa
      a ser rejeitada depois disso.
- [ ] Após 5 tentativas falhas consecutivas, o evento aparece em `webhook_dead_letters` e deixa de constar
      como `PENDING` na outbox ativa.
- [ ] `POST /api/v1/admin/webhooks/dead-letter/:id/replay` retorna `403 FORBIDDEN` para usuários com role
      `OPERATOR` e `202 Accepted` para `ADMIN`.
- [ ] URLs não-HTTPS são rejeitadas na criação/edição com `WEBHOOK_INVALID_URL`.
- [ ] Payload de evento acima de 64KB não gera linha na outbox e é registrado via log com motivo
      `WEBHOOK_PAYLOAD_TOO_LARGE`.
- [ ] Evento só é inserido na outbox para webhooks cujo `events` inclua o `toStatus` da transição.
- [ ] Testes end-to-end cobrem: cadastro → mudança de status filtrada → tentativa de entrega →
      sucesso registrado no histórico; e o caminho equivalente até a DLQ e o replay administrativo.

## Riscos e Mitigação

- **Contenção na tabela `webhook_outbox_events` entre inserts (`changeStatus`) e leituras do worker
  (polling).** Mitigação: índices em `status` e `createdAt`, leitura em lote pequeno, e uso de
  `SELECT ... FOR UPDATE SKIP LOCKED` (suportado pelo MySQL 8, já usado pelo projeto via
  `docker-compose.yml`) para reduzir contenção de lock durante o claim de eventos pelo worker.
- **Processamento duplicado caso, no futuro, mais de um worker suba simultaneamente.** Mitigação: o claim
  de eventos usa um `UPDATE ... WHERE status = 'PENDING'` atômico antes de processar, mesmo hoje sendo
  single-worker (ADR-005), tornando a migração futura para múltiplos workers menos arriscada.
- **Secret armazenada em texto plano no banco.** Risco aceito nesta fase; mitigado operacionalmente pela
  revisão de segurança dedicada antes do deploy (pelo menos 2 dias úteis reservados, com foco em HMAC e
  geração de secret).
- **Crescimento da fila de pendentes sem o time perceber (worker travado ou caído silenciosamente).**
  Mitigação: métrica `webhook_outbox_pending_count` disponível para alerta operacional (definição de
  limiares de alerta fica fora do escopo deste documento).

## Integração com o Sistema Existente

1. **`src/modules/orders/order.service.ts`** — `OrderService.changeStatus` é estendido: dentro do mesmo
   `prisma.$transaction` já existente, logo após `tx.orderStatusHistory.create(...)`, passa a chamar
   `publishWebhookEvent(tx, order, from, to)`. Essa função recebe o `tx` (Prisma `TransactionClient`) da
   transação em andamento — não o `WebhookRepository` inteiro injetado no `OrderService` — mantendo-a
   como uma função pura de publicação, sem acoplar o `OrderService` à camada de persistência do módulo
   de webhooks.
2. **`src/shared/errors/http-errors.ts` e `src/shared/errors/app-error.ts`** — as novas classes de erro do
   módulo (`WebhookNotFoundError`, `InvalidWebhookUrlError`, `WebhookPayloadTooLargeError`, etc.) estendem
   `AppError` exatamente como `InsufficientStockError` e `InvalidStatusTransitionError` já fazem hoje,
   reaproveitando o mesmo construtor `(message, statusCode, errorCode, details)`.
3. **`src/middlewares/error.middleware.ts`** — não precisa de nenhuma alteração: qualquer erro que estenda
   `AppError` já é capturado pelo bloco `if (err instanceof AppError)` existente e serializado no formato
   `{ error: { code, message, details } }` padrão do projeto.
4. **`src/middlewares/auth.middleware.ts`** — o endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
   reaproveita `requireRole('ADMIN')`, já exportado por esse middleware e usado da mesma forma que
   qualquer outra rota administrativa do projeto precisaria.
5. **`src/routes/index.ts` e `src/app.ts`** — `buildApiRouter` passa a montar
   `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` (e o sub-caminho administrativo),
   e `buildControllers` em `src/app.ts` instancia `WebhookRepository` → `WebhookService` →
   `WebhookController` da mesma forma manual que já faz para `orders`, `customers` e `products`.
6. **`src/shared/logger/index.ts`** — reaproveitado sem nova instância; única mudança pontual sugerida é
   adicionar `'*.secret'` à lista `redactPaths` já existente, para cobrir os novos payloads que carregam
   secret de webhook.
7. **`prisma/schema.prisma`** — recebe os novos models (`WebhookEndpoint`, `WebhookOutboxEvent`,
   `WebhookDeadLetter`, `WebhookDelivery`), seguindo o mesmo padrão dos models existentes (`id String @id
   @default(uuid()) @db.Char(36)`, `createdAt`/`updatedAt`, `@@map` em snake_case, índices explícitos em
   colunas usadas para filtro).
8. **`src/server.ts`** — serve de modelo direto para o novo entry-point `src/worker.ts`: mesmo padrão de
   bootstrap, tratamento de `SIGINT`/`SIGTERM` e desconexão graciosa do Prisma ao encerrar.
