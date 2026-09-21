# ADR-005: Worker de Entrega em Processo Separado com Polling de 2 Segundos

## Status

Aceita

## Contexto

Alguém precisa consumir a `webhook_outbox_events` (ADR-001) e efetivamente chamar os endpoints dos clientes.
A equipe discutiu se esse consumidor deveria rodar dentro do mesmo processo da API ou como um processo
independente, e como ele deveria descobrir novos eventos.

## Decisão

Criar um novo entry-point, `src/worker.ts`, seguindo o mesmo padrão de processo dedicado que
`src/server.ts` já estabelece para a API — porém rodando como **processo Node separado**, iniciado por
um script próprio (`npm run worker`). O worker conecta ao mesmo banco (`DATABASE_URL`) mas instancia seu
**próprio `PrismaClient`**, já que `PrismaClient` é por processo. O consumo é por **polling**: a cada 2
segundos, o worker busca os eventos pendentes mais antigos em lote pequeno, processa e marca como
entregues.

## Alternativas Consideradas

1. **Trigger de banco de dados para notificar o worker reativamente.** Descartada: MySQL não tem um
   mecanismo equivalente ao `LISTEN`/`NOTIFY` do Postgres; um trigger só executa SQL, não consegue
   notificar um processo externo sem soluções improvisadas (escrever em arquivo, chamar um endpoint).
2. **Rodar o worker dentro da mesma instância/processo da API.** Descartada: se a API reiniciar (deploy,
   crash, etc.), o worker cairia junto e pararia de processar a outbox.

## Consequências

**Positivas**
- Polling de 2s atende com folga o requisito de latência percebida pelos clientes ("abaixo de 10
  segundos é tempo real") — latência mínima de 2s no pior caso, aceita explicitamente pela equipe.
- Isolamento operacional: reiniciar a API não afeta o worker, e vice-versa.

**Negativas**
- Ordering de eventos só é garantida por `order_id`, e apenas enquanto houver um **único worker** ativo
  processando em ordem de `created_at`; não há garantia de ordering global entre pedidos diferentes.
  Escalar para múltiplos workers exigiria particionamento por `order_id` ou lock pessimista —
  explicitamente deixado como problema futuro, não desta feature.
