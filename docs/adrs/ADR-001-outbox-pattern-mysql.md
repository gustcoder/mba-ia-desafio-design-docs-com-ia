# ADR-001: Outbox Pattern no MySQL para Entrega de Eventos de Webhook

## Status

Aceita

## Contexto

O Sistema de Webhooks de Notificação de Pedidos precisa emitir um evento sempre que o status de um
pedido muda, para que os clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) parem de fazer
polling em `GET /orders`. A mudança de status hoje já roda dentro de uma transação SQL única em
`OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que atualiza `order`, insere em
`order_status_history` e ajusta `stockQuantity` dos produtos.

A equipe discutiu se o evento deveria ser disparado de forma síncrona (chamada HTTP direta dentro dessa
transação) ou assíncrona via algum mecanismo de fila/outbox.

## Decisão

Adotar o padrão Outbox sobre o MySQL existente: dentro da mesma transação que atualiza `order` e
`order_status_history`, inserir uma linha em uma nova tabela `webhook_outbox` com o evento já renderizado
(snapshot no momento da inserção, não recalculado depois). A tabela tem índice nos campos de status do
evento (pendente, processando, falhou, entregue) e em `created_at`. Um worker externo (ver ADR-005) lê os
eventos pendentes e faz a entrega.

Se a transação principal (`changeStatus`) commitar, o evento existe; se ela sofrer rollback, o evento
desaparece junto — não há inconsistência possível entre estado do pedido e evento enfileirado.

## Alternativas Consideradas

1. **Disparo síncrono da chamada HTTP dentro de `changeStatus`.** Descartada: um cliente lento ou fora
   do ar travaria a mudança de status de outros pedidos, e não há como dar rollback só da notificação
   sem também desfazer a transação de negócio.
2. **Fila externa dedicada (ex.: Redis Streams).** Descartada: exigiria subir e operar infraestrutura
   nova para um time pequeno; overengineering frente ao MySQL já disponível.

## Consequências

**Positivas**
- Atomicidade garantida entre mudança de status e enfileiramento do evento, sem infraestrutura nova.
- Reaproveita o banco e o `PrismaClient` já usados pelo restante da aplicação.

**Negativas**
- Acopla a emissão de eventos ao MySQL (sem pub/sub nativo); a leitura depende de polling (ADR-005).
- Requer rotina futura de arquivamento de eventos entregues (linhas entregues após ~30 dias), explicitamente
  deixada fora do escopo desta feature.
