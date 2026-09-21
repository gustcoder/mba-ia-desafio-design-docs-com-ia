# ADR-003: Autenticação de Webhooks via HMAC-SHA256 com Secret por Endpoint

## Status

Aceita

## Contexto

Os eventos de webhook carregam dados de pedidos e saem da infraestrutura da empresa para sistemas de
terceiros. O cliente precisa de uma forma de confirmar que a requisição veio realmente da plataforma e
que o payload não foi adulterado no caminho. A equipe já teve um incidente em que um cliente vazou uma
secret em log de aplicação, o que motivou também uma exigência de rotação.

## Decisão

Assinar o corpo de cada requisição de webhook com HMAC-SHA256, enviando a assinatura no header
`X-Signature`. Cada endpoint de webhook cadastrado por um customer tem sua **própria secret**, gerada
pela plataforma (não uma secret global compartilhada). A secret é rotacionável via API; ao rotacionar, a
secret antiga permanece válida em paralelo por 24 horas (grace period) para o cliente migrar seus
sistemas, e depois disso é invalidada.

## Alternativas Consideradas

1. **Secret global única para todos os endpoints da plataforma.** Descartada: um vazamento comprometeria
   a autenticidade de todos os webhooks de todos os clientes ao mesmo tempo — "se vaza uma, vaza tudo".
2. **Secret estática, sem suporte a rotação.** Descartada implicitamente: sem rotação, um vazamento de
   secret (cenário já observado com um cliente real) obrigaria a interromper a integração até recadastrar
   um novo endpoint, em vez de rotacionar com um período de transição seguro.

## Consequências

**Positivas**
- Segue um padrão amplamente adotado no mercado (HMAC-SHA256), com bibliotecas de verificação disponíveis
  para praticamente qualquer stack do lado do cliente.
- Isola o "raio de explosão" de um vazamento de secret a um único endpoint/cliente.
- Rotação com grace period de 24h permite recuperação segura sem downtime da integração.

**Negativas**
- O verificador de assinatura no worker precisa validar contra duas secrets simultaneamente durante a
  janela de rotação (a atual e a anterior, se ainda dentro das 24h).
- Mais estado por webhook a persistir e expirar (secret atual, secret anterior, timestamp de expiração
  da secret anterior).
