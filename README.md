# 🚀 Desafio MBA Engenharia de Software com IA - Full Cycle

![Status](https://img.shields.io/badge/Status-Em_Andamento-orange?style=for-the-badge&logo=github)
![IA](https://img.shields.io/badge/Focus-AI%20Engineering-blueviolet?style=for-the-badge&logo=openai)
![FullCycle](https://img.shields.io/badge/School-FullCycle-yellow?style=for-the-badge)


## Sobre o desafio
Primeiro gerei o conhecimento do projeto no Claude com o famigerado `/init`. Em seguida, utilizando o SDD, criei uma `context/intent` (conceito do framework ContextMesh) para elaborar um documento base que orientasse a criação das docs, aproveitando parte da estrutura do próprio desafio e adaptando-a conforme minhas ideias, sempre respeitando a proposta inicial. A partir daí, extraí os trechos de requisitos referentes a Skills, para usufruir desse recurso do Claude de forma progressiva — e também como exercício prático.

Com isso, já tinha em mãos um start consistente para o desafio, restando iterar quantas vezes fosse necessário até atingir o resultado esperado. Para isso, criei intents de revisão (`context/intent/review`), pautados nas exigências do desafio, que me ajudaram a validar os critérios obrigatórios. Por fim, redigi o README, trazendo a experiência final de todo o processo.

## Ferramentas de IA utilizadas
- **Claude CLI:** usado para levantar os requisitos do projeto e executar skills para criação dos documentos
- **Claude CoWork:** usado via chat para realizar ajustes finos que não necessitavam ser uma Spec
- **Context Mesh:** framework usado para organizar e estruturar as specs (SDD)

## Workflow adotado
1. Inicialização do projeto 
2. Definição do contexto (SDD + ContextMesh)
3. Extração de Skills
4. Iteração
5. Revisão via intents/specs (+ iterações)
6. Documentação final + checklist de critérios de aceite


## Prompts customizados
1. Optei por transformar a sessão de Requisitos das docs em skills na Spec criada no Context Mesh:
```
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
```

2. Adicionar conhecimento sobre as skills no CLAUDE.md:
```
Atualize o CLAUDE.md para conter as skills criadas no projeto, com instruções básicas de uso baseada no que está escrito na sessão "Requisitos" do arquivo de intent `docs-creator-v1`
```

## Iterações e ajustes
Ao revisar as documentações foi possível notar que várias menções diretas da transcrição, inclusive com timestamp [hh:ss] estavam presentes, dando a impressão de "copia e cola"
que queremos evitar.

Foi então aplicado um ajuste de **limpeza + melhoria de texto** nesse sentido:
```
 Remover os [hh:mm] Nome do corpo de PRD/RFC/FDD/ADRs (reescrevendo essas frases em prosa direta) e deixar toda a rastreabilidade formal apenas no Tracker
```

Ao revisar os critérios de aceite, foi possível identificar um possível gap de interpretação referente ao **PRD — "Fora de escopo"** ao rodar agentes automatizados.
```
PRD — "Fora de escopo" é um rótulo em negrito dentro de ## Escopo, não um heading ## próprio. 
O conteúdo satisfaz o critério (5 itens listados), mas se algum verificador automatizado procurar literalmente por um heading 
## Fora de Escopo, ele não vai encontrar.
```
Com isso realizei o ajuste pontual para manter a consistência.

Além disso também foram encontradas algumas discrepâncias no quesito padronização e consistência em uma revisão final, tais como:
1. Nome da tabela de outbox/DLQ diverge entre os documentos (a mais notável)
2. docs/adrs/README.md documenta uma convenção de nome errada
3. WEBHOOK_ALREADY_PROCESSED referencia um estado que a modelagem não guarda

Prompt proposto para ajuste:
```
padronizar em webhook_outbox_events/webhook_dead_letters (o nome plural do FDD) em todos os lugares, atualizar o docs/adrs/README.md para refletir a convenção ADR-NNN-*, e adicionar um campo replayedAt ao WebhookDeadLetter no FDD (ajustando a descrição do erro).
```


Ao todo foram necessárias **7 iterações principais** (considerando as specs de review) até chegar ao resultado final.

## Como navegar a entrega
1. `README.md`: consolidação das principais ideias utilizadas no desafio
2. `CLAUDE.md`: instruções gerais sobre o projeto e orientações sobre o uso de skills
3. `context/intent/docs-creator-v1.md`: spec inicial criada para conduzir o Claude na geração dos documentos
4. `context/intent/review`: specs adicionais para corrigir erros e aplicar ajustes finos
5. `.claude/skills/*-creator/SKILL.md`: instruções extraídas das definições do desafio para gerar skills visando o uso progressivo pela IA
6. `docs/adrs`: ADRs geradas para a feature
7. `docs/FDD.md`: Documento FDD gerado para a feature
8. `docs/PRD.md`: Documento PRD gerado para a feature
9. `docs/RFC.md`: Documento RFC gerado para a feature
10. `docs/TRACKER.md`: Documento TRACKER gerado para a feature

## Ideias Descartadas da Feature

| # | Ideia descartada/adiada | Quando/quem |
|---|---|---|
| 1 | Disparo síncrono da notificação dentro de `changeStatus` | [09:03]–[09:04] Bruno/Larissa |
| 2 | Fila externa dedicada (Redis Streams) em vez de outbox no MySQL | [09:06]–[09:07] Diego/Larissa |
| 3 | Trigger de banco (MySQL) para notificar o worker reativamente | [09:09] Diego |
| 4 | Retry indefinido (sem teto de tentativas) | [09:15] Diego |
| 5 | Retry com apenas 3 tentativas | [09:16] Bruno/Diego |
| 6 | Marcar falha permanente na própria `outbox` (`status = failed`) em vez de tabela DLQ separada | [09:18] Diego |
| 7 | Secret global compartilhada entre todos os endpoints (em vez de secret por endpoint) | [09:21] Sofia |
| 8 | Truncar payload acima do limite (em vez de erro) | [09:23]–[09:24] Sofia/Diego |
| 9 | Garantia de entrega exactly-once | [09:24]–[09:25] Diego |
| 10 | Notificação proativa (e-mail) ao cliente em falhas recorrentes | [09:37]–[09:38] Marcos/Larissa |
| 11 | Rate limiting de envio implementado nesta fase | [09:38]–[09:39] Diego/Larissa |
| 12 | Dashboard/painel visual para o cliente | [09:39]–[09:40] Larissa/Marcos |
| 13 | Múltiplos workers em paralelo com garantia de ordering global | [09:12]–[09:13] Diego |
| 14 | Webhooks de entrada (cliente enviando dados para nós) | [09:02]–[09:03] Marcos/Sofia |
| 15 | ID auto-incremental para a outbox (em vez de UUID) | [09:51] Larissa/Diego |
| 16 | Payload renderizado só na hora do envio, guardando apenas `order_id` (em vez de snapshot na inserção) | [09:51]–[09:52] Bruno/Larissa/Diego |

## Critérios de Aceite

**PRD (`docs/PRD.md`)**
- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias listadas no requisito 1
- [x] Identifica no mínimo 8 requisitos funcionais discutidos na reunião
- [x] Inclui pelo menos 1 objetivo com métrica e meta quantitativa
- [x] Seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião
- [x] Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação

**RFC (`docs/RFC.md`)**
- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias listadas no requisito 2
- [x] Seção "Alternativas consideradas" lista pelo menos 2 alternativas descartadas na reunião, cada uma com o trade-off que motivou o descarte
- [x] Seção "Questões em aberto" lista pelo menos 2 pontos adiados ou não decididos na reunião
- [x] Referencia, com link, pelo menos 2 ADRs do pacote

**FDD (`docs/FDD.md`)**
- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias listadas no requisito 3
- [x] Seção "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes
- [x] Matriz de erros usa códigos com prefixo WEBHOOK_
- [x] Seção "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais do código base
- [x] Seção "Observabilidade" cita métricas, logs e tracing

**ADRs (`docs/adrs/ADR-NNN-*.md`)**
- [x] Pasta docs/adrs/ contém entre 5 e 8 arquivos no formato ADR-NNN-titulo-em-kebab-case.md
- [x] Cada ADR contém as seções Status, Contexto, Decisão, Alternativas Consideradas, Consequências
- [x] O conjunto cobre pelo menos 5 das 6 decisões principais listadas no requisito 4
- [x] Pelo menos 1 ADR referencia explicitamente arquivos, módulos ou classes do código base

**Tracker (`docs/TRACKER.md`)**
- [x] Arquivo existe e segue o formato de tabela definido no requisito 5
- [x] Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente
- [x] Pelo menos 70% das linhas têm Fonte = TRANSCRICAO com timestamp válido no formato [hh:mm] Nome
- [x] Pelo menos 5 linhas têm Fonte = CODIGO com caminho de arquivo real

**README (`README.md`)**
- [x] Contém todas as seções obrigatórias listadas no requisito 6
- [x] Lista pelo menos 1 ferramenta de IA utilizada
- [x] Mostra pelo menos 2 prompts customizados em blocos de código
- [x] Descreve pelo menos 2 iterações ou ajustes concretos feitos durante a produção

**Consistência geral**
- [x] Nenhum requisito, decisão ou restrição registrada nos documentos contradiz a transcrição ou o código
- [x] Nenhum arquivo de código mencionado nos documentos é inexistente no repositório
