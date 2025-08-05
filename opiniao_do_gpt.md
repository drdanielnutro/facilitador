# Análise de Soluções - Agente ADK Flutter

## Validação Geral
As análises existentes cobrem bem os seis problemas, mas os itens 1 e 6 exigem refinamento para se alinharem melhor às práticas do ADK.

## Problema 1: Falta de Ferramentas no Orquestrador
### Validação da Análise Existente
Concordo que o `interactive_planner_agent` não possui meios de invocar os pipelines que coordena, apesar de listá-los em `sub_agents`【F:app copy/problemas.md†L5-L16】【F:app copy/comentarios_solucoes.md†L13-L16】.

### Soluções Alternativas
- Criar agentes *wrapper* para `planning_pipeline` e `execution_pipeline`, permitindo sua exposição como `AgentTool`.
- Substituir o `interactive_planner_agent` por um `BaseAgent` que chame os pipelines programaticamente via `EventActions`.

### Solução Recomendada
Os *wrappers* mantêm a simplicidade do `LlmAgent` e aproveitam o padrão já usado no agente funcional (`tools=[AgentTool(plan_generator)]`)【F:app/agent.py†L396-L408】.

### Implementação Concreta
```python
planning_invoker = LlmAgent(
    name="planning_invoker",
    instruction="Execute the planning pipeline",
    sub_agents=[planning_pipeline],
)

execution_invoker = LlmAgent(
    name="execution_invoker",
    instruction="Execute the execution pipeline",
    sub_agents=[execution_pipeline],
)

interactive_planner_agent = LlmAgent(
    ...,
    tools=[AgentTool(planning_invoker), AgentTool(execution_invoker)],
)
```

## Problema 6: Bootstrap do Sistema
### Validação da Análise Existente
O sistema depende de estado pré-existente e não há orquestrador capaz de conduzir a troca de fases【F:app copy/problemas.md†L138-L160】【F:app copy/comentarios_solucoes.md†L159-L168】.

### Soluções Alternativas
- Estado controlado por um `BaseAgent` com máquina de estados simples.
- Agente inicial dedicado à recepção de documentos e `feature_snippet`, delegando aos pipelines apenas quando os pré-requisitos forem atendidos.

### Solução Recomendada
Implementar um orquestrador derivado de `BaseAgent` que gerencie explicitamente as fases (documentos → feature → planejamento → execução).

### Implementação Concreta
```python
class FeatureOrchestrator(BaseAgent):
    def __init__(self):
        super().__init__(name="feature_orchestrator")
        self.phase = "waiting_docs"

    async def _run_async_impl(self, ctx: InvocationContext):
        state = ctx.session.state
        if self.phase == "waiting_docs":
            if self._docs_ready(state):
                self.phase = "waiting_feature"
                yield Event(author=self.name, content="Documentos recebidos. Envie a feature." )
            else:
                yield Event(author=self.name, content="Aguardando documentos...")
        elif self.phase == "waiting_feature":
            if state.get("feature_snippet"):
                self.phase = "planning"
                yield Event(author=self.name, actions=EventActions(transfer_to="planning_pipeline"))
        elif self.phase == "planning" and state.get("plan_approved"):
            self.phase = "executing"
            yield Event(author=self.name, actions=EventActions(transfer_to="execution_pipeline"))
```

## Problemas 2-5
- **Problema 2:** Usar `{{ feature_snippet? }}` e validar entrada inicial evita falhas de estado【F:app copy/problemas.md†L30-L51】.
- **Problema 3:** Tratar documentos como opcionais e ajustar instruções dinamicamente é suficiente【F:app copy/problemas.md†L55-L84】.
- **Problema 4:** Alterar o callback para `generated_code` resolve a coleta de trechos【F:app copy/problemas.md†L88-L107】.
- **Problema 5:** Calcular `total_tasks` e extrair `feature_name` do `implementation_plan` garante consistência no `SessionState`【F:app copy/problemas.md†L111-L133】.

## Conclusão e Priorização
1. **Problema 6:** sem bootstrap o sistema não inicia; implementar um `FeatureOrchestrator` é prioritário.
2. **Problema 1:** sem ferramentas o orquestrador não delega; adicionar *wrappers* para pipelines é a próxima etapa crítica.
3. Em seguida, corrigir o callback (Problema 4) e garantir `feature_snippet` (Problema 2).
4. Por fim, robustecer variáveis de estado e documentos (Problemas 5 e 3).

