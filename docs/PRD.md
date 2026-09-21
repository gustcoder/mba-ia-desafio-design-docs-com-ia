# PRD: Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature

O OMS (Order Management System) hoje não possui nenhum mecanismo de notificação externa, eventos, filas
ou webhooks — clientes integrados via API só descobrem mudanças no status de seus pedidos consultando
repetidamente `GET /orders`. Esta feature introduz um sistema de **webhooks outbound**: sempre que o
status de um pedido muda, os clientes que se inscreveram para aquele status recebem uma notificação HTTP
autenticada, assíncrona e resiliente a falhas temporárias, eliminando a necessidade de polling.

## Problema e Motivação

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — formalizaram um pedido para serem
notificados em tempo real quando o status de seus pedidos muda. Hoje eles fazem polling em `GET /orders`,
o que torna a integração deles lenta e cara. A Atlas Comercial sinalizou risco de migrar para um
concorrente caso a feature não seja entregue até o fim do trimestre — há, portanto, risco comercial
concreto associado ao atraso desta entrega.

## Público-Alvo e Cenários de Uso

- **Clientes B2B integrados via API** (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo):
  cadastram um ou mais webhooks para seus pedidos e passam a receber notificações automáticas de mudança
  de status, em vez de fazer polling.
- **Operadores/administradores internos do OMS** (usuários autenticados via JWT, mesma base de usuários
  já existente): cadastram/gerenciam a configuração de webhook em nome do customer, pois o cadastro é
  feito pela nossa API autenticada com o JWT do sistema interno, não pelo cliente diretamente.
- **Administradores (role `ADMIN`)**: atuam no cenário de exceção — reprocessar manualmente eventos que
  falharam permanentemente (Dead Letter Queue).

## Objetivos e Métricas de Sucesso

- Eliminar a necessidade de polling em `GET /orders` pelos clientes B2B integrados, entregando notificação
  de mudança de status em **até 10 segundos** na grande maioria dos casos — o limiar que os próprios
  clientes definiram como "tempo real". **Meta quantitativa:** latência ponta a ponta (mudança de status →
  tentativa de entrega) abaixo de 10 segundos em pelo menos 95% dos eventos em condições normais de
  operação.
- Reter os clientes B2B que sinalizaram risco de churn (em particular a Atlas Comercial), entregando a
  feature dentro do prazo comercial combinado (fim do trimestre).
- Garantir que nenhuma mudança de status de pedido "perca" seu evento correspondente por falha do sistema
  de notificação (atomicidade entre a transação de negócio e o registro do evento).

## Escopo

### Incluso

- CRUD de configuração de webhook por customer (URL, secret gerada pela plataforma, lista de status
  assinados, ativo/inativo).
- Rotação de secret self-service, com grace period de 24h para a secret anterior.
- Emissão automática de evento de notificação a cada mudança de status de pedido que corresponda aos
  status assinados por algum webhook do customer.
- Reentrega automática com backoff exponencial em caso de falha temporária do endpoint do cliente.
- Dead Letter Queue para falhas permanentes, com reprocessamento manual restrito a administradores.
- Histórico consultável das últimas 100 entregas por webhook (sucesso/falha, payload, resposta, tempo de
  resposta).
- Autenticação das entregas via HMAC-SHA256 por endpoint, com URL obrigatoriamente HTTPS.
- Garantia de entrega at-least-once, com identificador único por evento para deduplicação do cliente.

### Fora de Escopo

Itens explicitamente descartados ou adiados na reunião:

1. **Notificação proativa (ex.: e-mail) ao cliente quando o webhook dele falha repetidamente.** Descartada
   para esta fase; ficou registrada como possível item de uma fase futura, após medir o impacto da feature
   atual.
2. **Painel visual (dashboard) para o cliente acompanhar seus webhooks.** Descartado para esta fase — é um
   projeto separado, de responsabilidade do time de frontend.
3. **Rate limiting de envio de notificações por cliente.** Não incluído nesta fase; a equipe decidiu
   apenas observar o comportamento em produção e revisitar depois se necessário.
4. **Suporte a múltiplos workers processando a outbox em paralelo (com garantia de ordering global).**
   Fora de escopo — hoje assume-se um único worker; escalar exigiria particionamento por `order_id` ou
   lock pessimista, tratado como problema futuro.
5. **Entrada de webhooks (o cliente enviando dados para nós).** Escopo é estritamente outbound — a
   plataforma só envia notificações, não recebe dados de terceiros.

## Requisitos Funcionais

| ID | Requisito |
|---|---|
| FR-01 | O sistema permite cadastrar um webhook para um customer informando URL e a lista de status de pedido (`events`) que deseja receber. |
| FR-02 | A secret do webhook é gerada automaticamente pela plataforma na criação e devolvida ao chamador apenas nesse momento. |
| FR-03 | O sistema permite editar um webhook cadastrado (URL, `events` assinados, ativo/inativo). |
| FR-04 | O sistema permite remover um webhook cadastrado. |
| FR-05 | O sistema permite listar os webhooks cadastrados para um customer. |
| FR-06 | O sistema envia automaticamente uma notificação para cada webhook assinante sempre que o pedido correspondente muda para um status que aquele webhook assina. |
| FR-07 | O sistema permite consultar o histórico das últimas 100 entregas de um webhook (sucesso/falha, payload, resposta, tempo de resposta). |
| FR-08 | O sistema reenvia automaticamente eventos que falharem na entrega, com backoff exponencial, por até 5 tentativas. |
| FR-09 | Eventos que esgotam as tentativas de reenvio são registrados em uma fila de falhas permanentes (Dead Letter Queue) para investigação. |
| FR-10 | Um usuário com role `ADMIN` pode reprocessar manualmente um evento da Dead Letter Queue via endpoint dedicado. |
| FR-11 | Toda entrega de webhook é assinada com HMAC-SHA256 usando uma secret exclusiva daquele endpoint. |
| FR-12 | O sistema permite rotacionar a secret de um webhook via API, mantendo a secret anterior válida por 24h. |
| FR-13 | Cada evento entregue carrega um identificador único (`event_id`) para permitir deduplicação do lado do cliente. |

## Requisitos Não Funcionais

| ID | Requisito |
|---|---|
| NFR-01 | Latência entre a mudança de status e a primeira tentativa de entrega deve, no pior caso, ficar próxima do requisito de negócio de "abaixo de 10 segundos". |
| NFR-02 | URLs de webhook devem ser obrigatoriamente HTTPS; cadastro/edição com URL HTTP é rejeitado. |
| NFR-03 | O payload de um evento não pode exceder 64KB; eventos que ultrapassarem esse limite não são enviados. |
| NFR-04 | Toda chamada HTTP de entrega tem timeout de 10 segundos. |
| NFR-05 | A garantia de entrega é at-least-once (não exactly-once). |
| NFR-06 | A feature não deve introduzir infraestrutura nova (sem filas/broker externos); deve rodar sobre o MySQL/Prisma já existentes. |
| NFR-07 | O worker de entrega roda como processo separado da API, sobrevivendo a reinícios dela. |
| NFR-08 | Ordering de eventos é garantida apenas por `order_id`, e apenas enquanto houver um único worker ativo — limitação conhecida e aceita. |

## Decisões e Trade-offs Principais

- **Outbox no MySQL em vez de disparo síncrono ou fila externa (Redis).** Evita acoplar a transação de
  negócio à disponibilidade de um cliente externo, sem exigir infraestrutura nova — ver
  [ADR-001](adrs/ADR-001-outbox-pattern-mysql.md).
- **Retry com 5 tentativas e backoff de até 12h, com DLQ separada, em vez de retry indefinido ou 3
  tentativas.** Equilibra tolerância a indisponibilidades reais (já houve caso de horas) sem deixar
  eventos pendurados para sempre — ver [ADR-002](adrs/ADR-002-retry-backoff-dead-letter-queue.md).
- **Secret HMAC por endpoint, rotacionável, em vez de secret global.** Isola o impacto de um vazamento a
  um único cliente — ver [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md).
- **At-least-once com `X-Event-Id` em vez de exactly-once.** Reduz complexidade de implementação em troca
  de exigir deduplicação do lado do cliente — trade-off consciente, alinhado a padrão de mercado
  (Stripe, GitHub) — ver [ADR-004](adrs/ADR-004-at-least-once-x-event-id.md).
- **Worker single-instance em polling de 2s, processo separado da API.** Simplicidade operacional em
  troca de não garantir ordering global entre pedidos diferentes — ver
  [ADR-005](adrs/ADR-005-worker-separado-polling.md).
- **Reuso total dos padrões arquiteturais existentes** (módulos, `AppError`, Pino, error middleware) em
  vez de uma stack dedicada para o módulo novo — ver
  [ADR-006](adrs/ADR-006-reuso-padroes-existentes-do-projeto.md).

## Dependências

- Banco MySQL e `PrismaClient` já provisionados pelo projeto (`docker-compose.yml`, `prisma/schema.prisma`).
- Padrões de código já existentes: `AppError`, error middleware centralizado, logger Pino,
  `requireRole`, estrutura de módulos (`src/modules/*`).
- Disponibilidade da Engenharia de Segurança para revisão dedicada de pelo menos 2 dias úteis antes do
  deploy, com foco em HMAC e geração de secret.
- Confirmação e comunicação de prazo com a Atlas Comercial e demais clientes B2B, sob responsabilidade do
  Product Manager.

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Cliente mantém o endpoint indisponível além da janela total de retry (~15h) e perde eventos definitivamente | Média | Alto — perda de visibilidade do cliente sobre pedidos, risco comercial já sinalizado pela Atlas | DLQ preserva o evento para reprocessamento manual após o endpoint do cliente voltar a responder |
| Vazamento de secret do lado do cliente (já ocorreu antes com outro cliente) compromete a autenticidade das notificações daquele endpoint | Média (já ocorreu 1x) | Médio — impacto isolado ao endpoint daquele cliente, graças à secret por endpoint | Rotação de secret self-service com grace period de 24h, sem downtime da integração |
| Contenção/latência na tabela de outbox caso o volume de eventos cresça além do esperado | Baixa no curto prazo | Médio — atraso na entrega, risco de estourar o SLA percebido de "abaixo de 10s" | Índices dedicados (status, created_at), leitura em lote pequeno pelo worker, arquivamento futuro de eventos entregues (fora do escopo desta fase) |
| Atraso no prazo comercial (fim do trimestre) por dependência da revisão de segurança | Baixa | Alto — risco de perda do cliente Atlas Comercial | Revisão de segurança já reservada dentro da estimativa de 3 sprints, não deixada para o final sem previsão |

## Critérios de Aceitação

- [ ] Os clientes B2B identificados (Atlas Comercial, MaxDistribuição, Nova Cargo) conseguem cadastrar
      pelo menos um webhook e recebem notificação de mudança de status de um pedido assinado dentro do
      limiar de latência aceito por eles (~10 segundos), em condições normais de operação.
- [ ] Um cliente consegue consultar o histórico das últimas 100 entregas de um webhook seu.
- [ ] Uma indisponibilidade temporária do endpoint do cliente (dentro da janela de ~15h) é recuperada
      automaticamente via retry, sem intervenção manual.
- [ ] Falhas permanentes (após esgotar as tentativas) ficam disponíveis em uma fila de dead letter para
      reprocessamento manual por um administrador.
- [ ] Nenhuma URL de webhook não-HTTPS é aceita pelo sistema.
- [ ] A rotação de secret não interrompe a validação de assinaturas durante a janela de 24h de transição.

## Estratégia de Testes e Validação

- Testes de integração seguindo o padrão já usado no projeto (Vitest + banco MySQL real, ver
  `tests/setup.ts`), cobrindo: criação/edição/remoção de webhook, disparo de evento filtrado por
  `events` assinados, simulação de falha do endpoint do cliente até a Dead Letter Queue, replay
  administrativo e rotação de secret com a janela de 24h.
- Teste dedicado de atomicidade: forçar uma falha dentro da transação de `changeStatus` e confirmar que
  nenhum evento de webhook correspondente é inserido — não pode haver caso de status mudado sem evento
  gerado, nem evento gerado sem mudança de status efetivada.
- Revisão de segurança dedicada sobre a geração/assinatura HMAC e o fluxo de rotação de secret, antes do
  deploy em produção, com pelo menos 2 dias úteis reservados.
- Validação funcional direta com os três clientes B2B durante a fase de rollout, já que foram eles que
  originaram o requisito e definiram o limiar de latência aceitável.
