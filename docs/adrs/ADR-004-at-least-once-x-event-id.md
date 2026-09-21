# ADR-004: Garantia de Entrega At-Least-Once com Identificador X-Event-Id

## Status

Aceita

## Contexto

Com retries (ADR-002), o mesmo evento pode ser entregue mais de uma vez ao cliente — por exemplo, se a
entrega teve sucesso mas a confirmação (resposta HTTP) se perdeu antes do worker registrar o sucesso. A
equipe discutiu se deveria perseguir uma garantia de exactly-once ou aceitar at-least-once e colocar a
responsabilidade de deduplicação no cliente.

## Decisão

Garantir **at-least-once**: o cliente deve estar preparado para receber o mesmo evento mais de uma vez.
Cada evento recebe um `event_id` (UUID) gerado no momento em que é inserido na outbox, e esse mesmo
`event_id` é enviado no header `X-Event-Id` em todas as tentativas de entrega daquele evento (inclusive
retries). O cliente deduplica do lado dele usando esse identificador. Essa exigência será documentada de
forma destacada no portal de desenvolvedor para os clientes.

## Alternativas Consideradas

1. **Garantia exactly-once.** Descartada: exigiria coordenação bilateral (ex.: two-phase commit ou
   confirmação transacional do lado do cliente), com complexidade desproporcional ao ganho —
   at-least-once com `event_id` já resolve 99% dos casos práticos.
2. **Entregar eventos sem nenhum identificador único, deixando o cliente inferir duplicidade pelo
   conteúdo.** Não adotada: o padrão de mercado observado em provedores como Stripe e GitHub é fornecer
   um identificador explícito de evento para deduplicação, o que a equipe optou por seguir.

## Consequências

**Positivas**
- Implementação simples do lado da plataforma; não exige rastrear "confirmação de leitura" do cliente.
- Alinhado a um padrão de mercado já familiar para integradores B2B.

**Negativas**
- Transfere a responsabilidade de deduplicação para o cliente — exige documentação clara e pode gerar
  bugs do lado do integrador que não implementar a dedupe corretamente.
- Duplicidade real pode ocorrer em cenários de timeout do worker mesmo quando a entrega original foi
  bem-sucedida do lado do cliente.
