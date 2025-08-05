# Problemas Identificados no agent.py (app copy)

Este documento analisa os problemas encontrados no `/app copy/agent.py` em comparação com o `/app/agent.py` que funciona corretamente.

## Problema 1: Falta de Ferramentas no Orquestrador Principal

### Descrição
O `interactive_planner_agent` não possui ferramentas configuradas, impedindo-o de executar suas próprias tarefas.

### Arquivo Afetado
- **Arquivo**: `/app copy/agent.py`
- **Linha**: 835
- **Código**: `tools=[],`

### Por que Ocorre
O agente orquestrador precisa executar os pipelines (`planning_pipeline` e `execution_pipeline`), mas sem ferramentas, ele não consegue invocá-los diretamente. A instrução sugere que ele deveria "executar" os pipelines, mas não tem mecanismo para isso.

### Comparação com /app/agent.py
No arquivo original (linha 408):
```python
tools=[AgentTool(plan_generator)],
```
Isso permite que o `interactive_planner_agent` chame o `plan_generator` como ferramenta, iniciando o fluxo corretamente.

### Hipótese de Solução
Adicionar `AgentTool` para os pipelines ou criar agentes intermediários que possam ser chamados como ferramentas, similar ao padrão usado no `/app/agent.py`.

---

## Problema 2: Dependência de Variável Não Inicializada (feature_snippet)

### Descrição
O `context_synthesizer` espera receber `{feature_snippet}` mas não há garantia de que esta variável esteja no estado quando o agente é executado.

### Arquivo Afetado
- **Arquivo**: `/app copy/agent.py`
- **Linha**: 152
- **Código**: `{feature_snippet}`

### Por que Ocorre
O fluxo depende do `interactive_planner_agent` armazenar o `feature_snippet` no estado (mencionado na linha 776), mas:
1. Isso acontece apenas nas instruções, não há código que garanta
2. O `context_synthesizer` é executado pelo `planning_pipeline`, que pode ser chamado antes dessa inicialização

### Comparação com /app/agent.py
O `/app/agent.py` não depende de variáveis pré-existentes. O `plan_generator` (linha 191) usa `{{ research_plan? }}` com o operador opcional `?`, indicando que pode não existir inicialmente.

### Hipótese de Solução
1. Usar o operador opcional `{{ feature_snippet? }}` 
2. Ou garantir que o `feature_snippet` seja passado como parâmetro inicial ao executar o pipeline
3. Ou criar um agente inicial que capture e armazene o feature_snippet antes do pipeline

---

## Problema 3: Dependência de Documentos Externos

### Descrição
O sistema assume que três documentos específicos existem "no histórico da conversa", mas não há mecanismo para verificar ou lidar com sua ausência.

### Arquivo Afetado
- **Arquivo**: `/app copy/agent.py`
- **Linhas**: 154-158
- **Código**:
```
Você tem acesso aos seguintes documentos no histórico da conversa:
- `especificacao_tecnica_da_ui`: Arquitetura técnica do app Flutter
- `contexto_api`: Documentação da API backend
- `fonte_da_verdade_ux`: Especificação de UX/Design
```

### Por que Ocorre
O agente foi projetado assumindo um contexto específico que pode não existir. Não há validação ou fallback se os documentos não estiverem presentes.

### Comparação com /app/agent.py
O `/app/agent.py` é completamente auto-suficiente:
- Não depende de documentos pré-existentes
- Cria todo o contexto necessário através de pesquisas web
- O usuário fornece apenas o tópico de pesquisa

### Hipótese de Solução
1. Tornar os documentos opcionais com o `session-state.py` já criado
2. Adicionar validação no início do pipeline para verificar se os documentos existem
3. Criar um fluxo alternativo quando os documentos não estão disponíveis
4. Permitir que os documentos sejam fornecidos como parte do `feature_snippet`

---

## Problema 4: Callback com Nome de Variável Incorreto

### Descrição
O `collect_code_snippets_callback` procura por `current_task_code` que não é definido por nenhum agente.

### Arquivo Afetado
- **Arquivo**: `/app copy/agent.py`
- **Linha**: 65
- **Código**: `if "current_task_code" in callback_context.state:`

### Por que Ocorre
O callback espera uma chave que não existe. O `code_generator` salva seu output em `generated_code` (linha 397), não em `current_task_code`.

### Comparação com /app/agent.py
Os callbacks no `/app/agent.py` trabalham com dados padronizados:
- `collect_research_sources_callback` processa `grounding_metadata` de eventos
- `citation_replacement_callback` lê de `final_cited_report` que é definido por `output_key`

### Hipótese de Solução
Alterar o callback para procurar por `generated_code` em vez de `current_task_code`, ou garantir que o código seja copiado para a chave esperada antes do callback.

---

## Problema 5: Variáveis de Estado Não Definidas

### Descrição
Várias variáveis são esperadas no estado mas nunca são explicitamente definidas.

### Arquivo Afetado
- **Arquivo**: `/app copy/agent.py`
- **Linhas**: 
  - 544: `{feature_name}` - nunca definido
  - 319: `{total_tasks}` - precisa ser calculado

### Por que Ocorre
Os agentes esperam variáveis que deveriam ser definidas por outros agentes ou callbacks, mas não há garantia de que isso aconteça.

### Comparação com /app/agent.py
No `/app/agent.py`, todas as variáveis usadas são:
1. Definidas por `output_key` de agentes anteriores
2. Passadas explicitamente pelo usuário
3. Geradas internamente pelo próprio agente

### Hipótese de Solução
1. Usar o `SessionState` do `session-state.py` para garantir estrutura
2. Adicionar callbacks ou agentes intermediários para calcular valores derivados
3. Garantir que cada variável esperada tenha uma fonte clara de definição

---

## Problema 6: Bootstrap do Sistema

### Descrição
O sistema não tem um ponto de entrada claro - depende de estado pré-existente mas não define como esse estado é criado.

### Arquivo Afetado
- **Arquivo**: `/app copy/agent.py`
- **Linhas**: 762-769 (instruções do orchestrator)

### Por que Ocorre
O design assume um fluxo específico (receber documentos → receber feature → executar), mas não há código que implemente esse fluxo, apenas instruções em linguagem natural.

### Comparação com /app/agent.py
O `/app/agent.py` tem um fluxo claro:
1. Usuário fornece tópico
2. `plan_generator` cria plano (via AgentTool)
3. Pipeline executa após aprovação

### Hipótese de Solução
1. Criar um agente de "setup" que gerencie a recepção de documentos
2. Implementar um fluxo de inicialização explícito
3. Usar o `SessionState` para validar pré-condições antes de executar
4. Tornar o sistema mais flexível para funcionar com ou sem os documentos

---

## Resumo

Os problemas principais derivam de:
1. **Assumir contexto não garantido** - documentos e variáveis que podem não existir
2. **Falta de ferramentas adequadas** - orquestrador sem meios de executar suas tarefas
3. **Desalinhamento entre expectativas e implementação** - callbacks e variáveis com nomes incorretos

O `/app/agent.py` evita esses problemas sendo:
- Auto-suficiente
- Usando ferramentas corretamente (AgentTool)
- Não assumindo estado pré-existente
- Tendo um fluxo linear claro com responsabilidades bem definidas