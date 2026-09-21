# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| Autor | Time de Engenharia (consolidado a partir da reunião técnica conduzida por Larissa, Tech Lead) |
| Status | Em revisão |
| Data | 2026-09-21 |
| Revisores | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Eng. Pleno, time de Pedidos), Diego (Eng. Sênior, time de Plataforma), Sofia (Eng. de Segurança) |

## Resumo Executivo (TL;DR)

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) precisam saber, em até 10 segundos, quando
o status de um pedido muda, sem depender de polling em `GET /orders`. Propomos um sistema de **webhooks
outbound** (só saída, a plataforma nunca recebe webhooks de terceiros) baseado no **padrão Outbox sobre o
MySQL já existente**: a mudança de status insere um evento na mesma transação que hoje atualiza `order` e
`order_status_history` (`OrderService.changeStatus`), e um **worker em processo separado**, rodando em
polling de 2 segundos, processa a entrega com **retry exponencial**, **Dead Letter Queue (DLQ)** e
**autenticação HMAC-SHA256 por endpoint**. A garantia de entrega é **at-least-once**, com deduplicação do
lado do cliente via `X-Event-Id`. Nenhuma infraestrutura nova é introduzida (sem Redis, sem fila externa);
o módulo reaproveita os padrões já estabelecidos no código (estrutura de módulos, `AppError`, Pino, error
middleware).

## Contexto e Problema

Hoje, clientes B2B integrados via API só descobrem mudanças de status de seus pedidos consultando
periodicamente `GET /orders`. Isso torna a integração lenta e cara para eles, e a Atlas Comercial já
sinalizou risco de migrar para um concorrente se isso não for resolvido até o fim do trimestre. O
requisito de latência aceito pelos clientes é "abaixo de 10 segundos" — não é preciso push instantâneo,
mas o processo manual de polling precisa acabar.

A aplicação atual (`src/`) não possui nenhum mecanismo de notificação externa, eventos, filas ou
webhooks — este é o vácuo que a feature preenche. O ponto de integração central é
`OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que já executa, dentro de uma única
transação Prisma, a atualização de status, o registro de histórico e o ajuste de estoque.

## Proposta Técnica

A proposta tem quatro pilares, cada um formalizado em um ADR dedicado (seção "Decisões relacionadas"):

1. **Emissão transacional (Outbox).** Ao mudar o status de um pedido, `changeStatus` insere — na mesma
   transação SQL — um evento em uma tabela `webhook_outbox`, já com o payload renderizado (snapshot do
   momento da mudança, não recalculado depois). Se a transação principal falhar, o evento nunca existiu;
   se ela commitar, o evento está garantidamente lá. A inserção é filtrada pela lista de status que cada
   webhook do customer assinou — se nenhum webhook quer aquele status, nada é inserido.
2. **Entrega assíncrona por worker dedicado.** Um novo processo Node (`src/worker.ts`, entry-point
   paralelo a `src/server.ts`) faz polling a cada 2 segundos nos eventos pendentes mais antigos e chama o
   endpoint HTTP configurado pelo cliente, com timeout de 10 segundos por chamada.
3. **Resiliência via retry + DLQ.** Falhas de entrega são reentregues com backoff exponencial (5
   tentativas, 1m/5m/30m/2h/12h); esgotadas as tentativas, o evento vai para uma tabela
   `webhook_dead_letter` e pode ser reprocessado manualmente por um endpoint administrativo restrito a
   role `ADMIN`.
4. **Segurança da entrega.** Cada endpoint de webhook cadastrado tem uma secret própria (não global),
   usada para assinar o payload com HMAC-SHA256 (`X-Signature`), com suporte a rotação com grace period de
   24h. URLs de webhook só são aceitas em HTTPS. Todo evento carrega um `X-Event-Id` único, e a garantia de
   entrega é at-least-once — o cliente deduplica pelo `event_id`.

O módulo novo, `src/modules/webhooks`, segue a mesma estrutura controller/service/repository/routes/schemas
já usada pelos demais módulos (`orders`, `customers`, `products`, `users`), reaproveitando `AppError`,
o error middleware centralizado, o logger Pino e o middleware `requireRole` existentes — nenhuma dessas
peças compartilhadas precisa mudar. O detalhamento de contratos HTTP, matriz de erros, fluxos passo a passo
e observabilidade fica no FDD (`docs/FDD.md`); esta seção descreve apenas a abordagem, não a implementação.

## Alternativas Consideradas

1. **Disparo síncrono da notificação dentro de `changeStatus`.** Foi a primeira opção discutida.
   Descartada porque acoplaria a latência/disponibilidade de um cliente externo à transação de negócio:
   um cliente lento travaria mudanças de status de outros pedidos, e não haveria como fazer rollback só
   da notificação sem desfazer a transação inteira.
2. **Fila externa dedicada (ex.: Redis Streams) em vez de outbox no MySQL.** Chegou a ser cogitada como
   alternativa ao outbox. Descartada por exigir subir e operar infraestrutura nova para um time pequeno —
   considerada overengineering frente ao MySQL já disponível e suficiente para o volume esperado.
3. **Garantia de entrega exactly-once.** Cogitada implicitamente ao discutir duplicidade. Descartada por
   exigir coordenação bilateral complexa entre plataforma e cliente, com ganho marginal frente a
   at-least-once + `X-Event-Id` (padrão já usado por Stripe e GitHub), que resolve "99% dos casos".

## Questões em Aberto

1. **Rate limiting de saída para clientes com muitos eventos em pouco tempo.** Se um cliente tem, por
   exemplo, 50 pedidos mudando de status no mesmo minuto, o worker dispararia 50 chamadas seguidas para o
   endpoint dele. A equipe decidiu não implementar rate limiting nesta fase, apenas observar o
   comportamento em produção e decidir depois se vira um problema real.
2. **Notificação proativa de endpoints com falha recorrente.** Foi levantada a ideia de avisar o cliente
   (ex.: por e-mail) quando o webhook dele falha repetidamente. Explicitamente adiada para uma fase
   futura, após medir o impacto real da feature atual.
3. **Escalar o worker para múltiplas instâncias.** Hoje a proposta assume um único worker (necessário
   para a ordenação por `order_id`, ver ADR-005). Particionamento por `order_id` ou lock pessimista para
   permitir múltiplos workers foi identificado como "problema do futuro", sem desenho definido.

## Impacto e Riscos

- **Impacto no fluxo crítico de pedidos.** `OrderService.changeStatus` passa a ter mais uma escrita
  dentro da mesma transação (inserção na outbox). O risco é mitigado por ser apenas um `INSERT` na mesma
  transação já existente, sem chamada de rede síncrona — a decisão de não disparar HTTP nesse caminho
  (ver Alternativa 1) existe justamente para isolar esse risco.
- **Exposição de dados de pedidos para fora da infraestrutura.** Mitigado por HMAC-SHA256 por endpoint,
  TLS obrigatório e payload enxuto (sem `items`, apenas campos agregados como `total_cents`).
- **Vazamento de secret de cliente.** Já ocorreu uma vez (secret vazada em log de aplicação de um
  cliente). Mitigado por secret única por endpoint e suporte a rotação com grace period.
- **Ordering entre eventos do mesmo pedido.** Garantida apenas enquanto houver um único worker; não há
  garantia de ordering global entre pedidos diferentes. Aceito porque os clientes nunca pediram ordering
  global, só a atualização de cada pedido deles.
- **Prazo.** Estimativa de 3 sprints, incluindo revisão de segurança dedicada (pelo menos 2 dias úteis
  reservados antes do deploy, com foco em HMAC e geração de secret).

## Decisões Relacionadas

- [ADR-001 — Outbox Pattern no MySQL para Entrega de Eventos de Webhook](adrs/ADR-001-outbox-pattern-mysql.md)
- [ADR-002 — Retry com Backoff Exponencial e Dead Letter Queue](adrs/ADR-002-retry-backoff-dead-letter-queue.md)
- [ADR-003 — Autenticação de Webhooks via HMAC-SHA256 com Secret por Endpoint](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — Garantia de Entrega At-Least-Once com Identificador X-Event-Id](adrs/ADR-004-at-least-once-x-event-id.md)
- [ADR-005 — Worker de Entrega em Processo Separado com Polling de 2 Segundos](adrs/ADR-005-worker-separado-polling.md)
- [ADR-006 — Reuso dos Padrões Arquiteturais Existentes do Projeto](adrs/ADR-006-reuso-padroes-existentes-do-projeto.md)
