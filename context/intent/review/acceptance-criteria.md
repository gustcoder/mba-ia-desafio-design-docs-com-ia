## Objetivo
Avaliar as documentações geradas e presentes em `../../../docs` e cruzar com o checklist abaixo:

**PRD (`../../../docs/PRD.md`)**
- [ ] Arquivo existe e está em Markdown
- [ ] Contém todas as seções obrigatórias listadas no requisito 1
- [ ] Identifica no mínimo 8 requisitos funcionais discutidos na reunião
- [ ] Inclui pelo menos 1 objetivo com métrica e meta quantitativa
- [ ] Seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião
- [ ] Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação

**RFC (`../../../docs/RFC.md`)**
- [ ] Arquivo existe e está em Markdown
- [ ] Contém todas as seções obrigatórias listadas no requisito 2
- [ ] Seção "Alternativas consideradas" lista pelo menos 2 alternativas descartadas na reunião, cada uma com o trade-off que motivou o descarte
- [ ] Seção "Questões em aberto" lista pelo menos 2 pontos adiados ou não decididos na reunião
- [ ] Referencia, com link, pelo menos 2 ADRs do pacote
  FDD (`../../../docs/FDD.md`)
- [ ] Arquivo existe e está em Markdown
- [ ] Contém todas as seções obrigatórias listadas no requisito 3
- [ ] Seção "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes
- [ ] Matriz de erros usa códigos com prefixo WEBHOOK_
- [ ] Seção "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais do código base
- [ ] Seção "Observabilidade" cita métricas, logs e tracing

**ADRs (`../../../docs/adrs/ADR-NNN-*.md`)**
- [ ] Pasta docs/adrs/ contém entre 5 e 8 arquivos no formato ADR-NNN-titulo-em-kebab-case.md
- [ ] Cada ADR contém as seções Status, Contexto, Decisão, Alternativas Consideradas, Consequências
- [ ] O conjunto cobre pelo menos 5 das 6 decisões principais listadas no requisito 4
- [ ] Pelo menos 1 ADR referencia explicitamente arquivos, módulos ou classes do código base

**Tracker (`../../../docs/TRACKER.md`)**
- [ ] Arquivo existe e segue o formato de tabela definido no requisito 5
- [ ] Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente
- [ ] Pelo menos 70% das linhas têm Fonte = TRANSCRICAO com timestamp válido no formato [hh:mm] Nome
- [ ] Pelo menos 5 linhas têm Fonte = CODIGO com caminho de arquivo real

**Consistência geral**
- [ ] Nenhum requisito, decisão ou restrição registrada nos documentos contradiz a transcrição ou o código
- [ ] Nenhum arquivo de código mencionado nos documentos é inexistente no repositório

## Output esperado
- Lista com os critérios avaliados
- Caso algum item não seja contemplado, apontar quais são para que eu avalie e faça os ajustes necessários
