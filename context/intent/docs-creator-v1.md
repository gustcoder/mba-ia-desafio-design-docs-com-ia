# Contexto

Uma empresa que opera um Order Management System (OMS) em produção vai construir uma nova feature, um Sistema de Webhooks de Notificação de Pedidos. A decisão técnica já foi tomada em uma reunião entre tech lead, PM, engenheiros e segurança, mas nada foi registrado além da transcrição da call (`../../TRANSCRICAO.md`).

## Tarefa
Produzir, a partir da transcrição e do código existente, a documentação técnica da feature, em nível acionável o suficiente para o time de engenharia iniciar a implementação.


## Objetivo
Criação do seguinte pacote de documentações:

- PRD (Product Requirement Document) da feature
- RFC (Request for Comments) com a proposta técnica da solução, submetida à equipe para revisão
- FDD (Feature Design Document) da feature
- 6 ADRs (Architecture Decision Records) das decisões discutidas
- Tracker de rastreabilidade ligando cada item à origem na transcrição ou no código
- README atualizado documentando o processo de produção

Obs.: Toda informação registrada nos documentos deve ser rastreável à transcrição ou ao código fonte da aplicação. Não é permitido inventar requisitos, decisões ou restrições sem origem identificável.

## Definições das Documentações
1. Os documentos não podem se repetir, ou seja, cada um opera em uma altura diferente
2. Antes de produzir, entenda a fronteira entre eles: conteúdo duplicado entre documentos é sinal de que algo está no lugar errado.

| Documento | 	Papel                                                                                        | 	Altura               | 	Pergunta que responde                                    |
|-----------|----------------------------------------------------------------------------------------------|----------------------|----------------------------------------------------------|
| PRD       | 	Problema, público, escopo e métricas de sucesso                                              | Produto / negócio    | Por que e o quê?                                         |                                            
| RFC       | 	Proposta técnica da solução para revisão: abordagem geral, alternativas e questões em aberto | 	Arquitetura          | 	Como pretendemos resolver, e o que ainda está em aberto? |    
| ADRs      | 	Cada decisão arquitetural isolada, com contexto e consequências                              | 	Decisão pontual      | 	Por que decidimos exatamente assim?                      |                         
| FDD       | 	Especificação de implementação: fluxos, contratos, erros, integração com o código            | 	Implementação        | 	Como construir, em detalhe?                              |                                 
| Tracker   | 	Rastreabilidade de cada item ao código ou à transcrição                                      | 	Transversal          | 	De onde veio cada coisa?                                 |

### Requisitos
#### 1. PRD da feature
Usar a skill `prd-creator` cobrindo a feature de Sistema de Webhooks de Notificação de Pedidos.

#### 2. RFC da feature
Usar a skill `rfc-creator` para criar documento com a proposta técnica da solução, no formato de um documento submetido à equipe para revisão.

#### 3. FDD da feature
Usar a skill `fdd-creator` para detalhar o "como implementar" da feature.

#### 4. ADRs
Usando a skill `adr-creator`, produza 6 ADRs em arquivos separados dentro de `../../docs/adrs/`.

#### 5. Tracker de Rastreabilidade
Usar a skill `tracker-creator` para criar o arquivo de rastreabilidade.

#### 6. README com o processo
Será criado manualmente.

## Diretivas & Restrições

- A reunião descarta explicitamente algumas ideias (`../../TRANSCRICAO.md`). Sempre certifique-se de NÃO considera-las como requisitos nas docs.
- NÃO alterar o código da aplicação, apenas trabalhar na criação documental.
- Cuide para não gerar trechos vagos ou sem relevância. Utilize exemplos concretos.
- Identifique as intenções propostas no arquivo de transcrição para evitar propostas superficiais ou ser somente um "copia + cola" da transcrição
