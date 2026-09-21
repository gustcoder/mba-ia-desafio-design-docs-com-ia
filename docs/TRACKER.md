# Tracker de Rastreabilidade

Mapeia cada item registrado no pacote de documentação (PRD, RFC, FDD, ADRs) à sua origem na transcrição
da reunião (`TRANSCRICAO.md`) ou no código-fonte do repositório. Itens sem origem identificável foram
deliberadamente deixados fora desta tabela (ex.: nomes específicos de métricas propostas na seção de
Observabilidade do FDD, ou riscos técnicos de contenção de lock), conforme orientação do desafio: se não é
possível preencher a coluna "Localização" com uma fonte real, o item não deve ser rastreado como se
tivesse origem — ele permanece apenas como elaboração técnica no documento correspondente.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-GOAL-01 | docs/PRD.md | Requisito Não Funcional | Meta quantitativa: latência de entrega < 10s em ≥95% dos eventos | TRANSCRICAO | [09:02] Marcos |
| PRD-GOAL-02 | docs/PRD.md | Decisão | Objetivo de reter a Atlas Comercial entregando dentro do prazo do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook informando URL e lista de status assinados | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma na criação, não pelo cliente | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar webhook cadastrado (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover webhook cadastrado (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Envio automático de evento filtrado pelos status assinados | TRANSCRICAO | [09:33]–[09:34] Marcos/Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Consultar histórico das últimas 100 entregas de um webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Reenvio automático com backoff exponencial, até 5 tentativas | TRANSCRICAO | [09:15]–[09:17] Diego/Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Dead Letter Queue para falhas permanentes | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay manual de item da DLQ restrito a role ADMIN | TRANSCRICAO | [09:35]–[09:36] Larissa/Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 usando secret exclusiva por endpoint | TRANSCRICAO | [09:19]–[09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21]–[09:22] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | X-Event-Id único por evento para deduplicação do cliente | TRANSCRICAO | [09:24]–[09:25] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência ponta a ponta alinhada ao requisito de "abaixo de 10s" | TRANSCRICAO | [09:09]–[09:10] Diego/Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | URL de webhook obrigatoriamente HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Payload de evento limitado a 64KB | TRANSCRICAO | [09:23]–[09:24] Sofia/Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP de entrega | TRANSCRICAO | [09:42] Diego/Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once (não exactly-once) | TRANSCRICAO | [09:24]–[09:25] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Nenhuma infraestrutura nova (sem Redis/fila externa) | TRANSCRICAO | [09:06]–[09:07] Diego/Larissa |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Worker roda em processo separado da API | TRANSCRICAO | [09:11] Diego/Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordering garantida só por order_id e em regime single-worker | TRANSCRICAO | [09:12]–[09:13] Diego/Larissa |
| PRD-SCOPE-OUT-01 | docs/PRD.md | Restrição | Notificação por e-mail em falhas recorrentes adiada para fase futura | TRANSCRICAO | [09:37]–[09:38] Marcos/Larissa |
| PRD-SCOPE-OUT-02 | docs/PRD.md | Restrição | Dashboard visual para o cliente fora de escopo | TRANSCRICAO | [09:39]–[09:40] Larissa/Marcos |
| PRD-SCOPE-OUT-03 | docs/PRD.md | Restrição | Rate limiting de saída não implementado nesta fase | TRANSCRICAO | [09:38]–[09:39] Diego/Larissa |
| PRD-SCOPE-OUT-04 | docs/PRD.md | Restrição | Múltiplos workers em paralelo / ordering global fora de escopo | TRANSCRICAO | [09:12]–[09:13] Diego |
| PRD-SCOPE-OUT-05 | docs/PRD.md | Restrição | Escopo é estritamente outbound (não recebemos webhooks de clientes) | TRANSCRICAO | [09:02] Marcos/Sofia |
| PRD-RISK-01 | docs/PRD.md | Trade-off | Risco de perda de eventos se cliente ficar indisponível além da janela de retry (~15h) | TRANSCRICAO | [09:15]–[09:17] Diego |
| PRD-RISK-02 | docs/PRD.md | Trade-off | Risco de vazamento de secret do lado do cliente (incidente já ocorrido) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Trade-off | Risco de atraso de prazo por dependência da revisão de segurança da Sofia | TRANSCRICAO | [09:46]–[09:47] Larissa/Sofia |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo síncrono descartado: acopla latência/disponibilidade do cliente à transação de negócio | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Fila externa (Redis Streams) descartada por overengineering para o tamanho do time | TRANSCRICAO | [09:06]–[09:07] Diego/Larissa |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Garantia exactly-once descartada pela complexidade de coordenação bilateral | TRANSCRICAO | [09:24]–[09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Rate limiting de saída não decidido, apenas observar e revisitar depois | TRANSCRICAO | [09:38]–[09:39] Diego/Larissa |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Notificação proativa de falha (e-mail) não decidida, adiada | TRANSCRICAO | [09:37]–[09:38] Marcos/Larissa |
| RFC-OPEN-03 | docs/RFC.md | Restrição | Escalar para múltiplos workers sem desenho definido | TRANSCRICAO | [09:12]–[09:13] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-mysql.md | Decisão | Outbox no MySQL, evento inserido na mesma transação de changeStatus | TRANSCRICAO | [09:06]–[09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dead-letter-queue.md | Decisão | Retry 5x com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | TRANSCRICAO | [09:15]–[09:18] Diego/Larissa |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | TRANSCRICAO | [09:20]–[09:22] Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-x-event-id.md | Decisão | At-least-once com X-Event-Id para deduplicação do cliente | TRANSCRICAO | [09:24]–[09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-separado-polling.md | Decisão | Worker em processo separado (src/worker.ts), polling de 2s | TRANSCRICAO | [09:09]–[09:11] Diego/Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md | Decisão | Reuso dos padrões arquiteturais existentes (módulos, AppError, Pino, error middleware) | TRANSCRICAO | [09:27]–[09:30] Bruno/Larissa |
| ADR-006-CODIGO | docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md | Decisão | Novas classes WEBHOOK_* seguem o mesmo padrão de AppError já usado por InsufficientStockError/InvalidStatusTransitionError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-FLOW-01 | docs/FDD.md | Decisão | Criação do evento na outbox dentro da transação de changeStatus | TRANSCRICAO | [09:40]–[09:41] Bruno/Diego |
| FDD-FLOW-02 | docs/FDD.md | Decisão | Processamento pelo worker: polling, timeout de 10s, headers de entrega | TRANSCRICAO | [09:42]–[09:44] Diego/Sofia |
| FDD-FLOW-03 | docs/FDD.md | Decisão | Fluxo de retry com backoff exponencial | TRANSCRICAO | [09:15]–[09:17] Diego |
| FDD-FLOW-04 | docs/FDD.md | Decisão | Fluxo de Dead Letter Queue e replay administrativo | TRANSCRICAO | [09:18]–[09:19] Diego/Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks — cadastro de webhook | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks — listagem de webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id — edição de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id — remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret — rotação de secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay — replay de DLQ (ADMIN) | TRANSCRICAO | [09:18], [09:35] Diego/Larissa |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Headers de entrega: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44]–[09:45] Diego/Sofia |
| FDD-ERR-01 | docs/FDD.md | Restrição | Código de erro WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Restrição | Código de erro WEBHOOK_INVALID_URL | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Restrição | Código de erro WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Restrição | Prefixo WEBHOOK_ obrigatório para todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-05 | docs/FDD.md | Restrição | WEBHOOK_PAYLOAD_TOO_LARGE — regra de negócio de erro acima de 64KB | TRANSCRICAO | [09:23]–[09:24] Sofia/Diego |
| FDD-ERR-06 | docs/FDD.md | Restrição | WEBHOOK_INVALID_EVENT_FILTER segue o padrão de validação de enum já usado para status de pedido | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-INTEGR-01 | docs/FDD.md | Decisão | changeStatus estendido para chamar publishWebhookEvent(tx, ...) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEGR-01B | docs/FDD.md | Restrição | publishWebhookEvent recebe o tx client, não um repository inteiro injetado | TRANSCRICAO | [09:41] Bruno/Diego |
| FDD-INTEGR-02 | docs/FDD.md | Decisão | Erros do módulo estendem AppError, mesmo padrão de InsufficientStockError/InvalidStatusTransitionError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEGR-03 | docs/FDD.md | Decisão | Error middleware existente já captura qualquer AppError sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEGR-04 | docs/FDD.md | Decisão | Endpoint de replay reaproveita requireRole('ADMIN') existente | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEGR-05 | docs/FDD.md | Decisão | Novo router de webhooks montado em buildApiRouter/buildControllers | CODIGO | src/routes/index.ts |
| FDD-INTEGR-06 | docs/FDD.md | Decisão | Logger Pino existente reaproveitado; recomenda adicionar '*.secret' ao redact | CODIGO | src/shared/logger/index.ts |
| FDD-INTEGR-07 | docs/FDD.md | Decisão | Novos models Prisma seguem o padrão de UUID/@@map dos models existentes | CODIGO | prisma/schema.prisma |
| FDD-INTEGR-08 | docs/FDD.md | Decisão | src/worker.ts espelha o padrão de bootstrap/shutdown de src/server.ts | TRANSCRICAO | [09:11] Larissa/Diego |
| FDD-RISK-01 | docs/FDD.md | Trade-off | Risco de secret em texto plano mitigado por revisão de segurança dedicada antes do deploy | TRANSCRICAO | [09:46]–[09:47] Larissa/Sofia |

## Cobertura

- Total de linhas: 72.
- Linhas com Fonte = TRANSCRICAO: 63 (~87,5%).
- Linhas com Fonte = CODIGO: 9 (~12,5%), todas com caminho de arquivo real do repositório.
- Cobertura estimada dos itens identificáveis do pacote (PRD + RFC + FDD + ADRs): acima de 80% — os itens
  deixados de fora são exclusivamente elaborações técnicas sem origem verificável (nomes de métricas
  propostas, riscos de contenção de lock, e dois códigos de erro adicionais no FDD sem correspondência
  literal na transcrição ou no código), conforme a orientação do desafio de não forçar uma linha quando a
  coluna "Localização" não pode ser preenchida honestamente.
