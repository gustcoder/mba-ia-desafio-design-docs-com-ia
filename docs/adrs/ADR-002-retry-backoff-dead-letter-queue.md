# ADR-002: Retry com Backoff Exponencial e Dead Letter Queue

## Status

Aceita

## Contexto

Clientes B2B podem estar temporariamente indisponíveis (já houve caso de indisponibilidade planejada de
duas horas). O sistema precisa de uma estratégia de reentrega que não deixe eventos pendurados
indefinidamente, mas que também não desista cedo demais e perca notificações válidas.

## Decisão

Reentregar com backoff exponencial: 5 tentativas, com intervalos de 1 minuto, 5 minutos, 30 minutos,
2 horas e 12 horas entre elas (~15 horas entre a primeira falha e a última tentativa). Esgotadas as 5
tentativas, o evento é movido para uma tabela separada, `webhook_dead_letter`, contendo o payload, o
motivo da falha e o timestamp. O reprocessamento é manual, via
`POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente, e exige
role `ADMIN` com log de auditoria de quem executou o replay.

## Alternativas Consideradas

1. **Retry indefinido com backoff.** Descartada: um evento cujo cliente "sumiu" ficaria retentando para
   sempre, sem sinalização clara de falha permanente.
2. **3 tentativas.** Descartada por ser agressiva demais: em ~30 minutos as 3 tentativas se esgotariam,
   o que já derrubaria a notificação de um cliente com uma manutenção planejada de poucas horas.
3. **Marcar o evento como `failed` na própria tabela `webhook_outbox`, sem tabela separada.** Descartada
   em favor de uma tabela dedicada: mantém a outbox principal enxuta e a DLQ serve como evidência isolada
   para debug e reprocessamento.

## Consequências

**Positivas**
- Cobre janelas de indisponibilidade de até ~15 horas antes de desistir, considerado aceitável mesmo
  para falhas prolongadas do lado do cliente.
- DLQ auditável e reprocessável sem poluir a tabela operacional principal.

**Negativas**
- Mais uma tabela (`webhook_dead_letter`) e um endpoint administrativo adicionais para manter, com sua
  própria exigência de autorização (`requireRole('ADMIN')`) e auditoria.
- Eventos em retry ficam por até ~15 horas em estado "pendente/falhando", período em que o cliente já
  pode ter perdido a atualização em tempo hábil — risco aceito pela equipe.
