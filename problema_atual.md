# Análise do Problema de Execução no Pipeline de Agentes ADK

## 1. Resumo do Problema (TL;DR)

O pipeline de geração de código, orquestrado com Google ADK, está terminando prematuramente. O loop principal, responsável por iterar sobre uma lista de tarefas de codificação (`task_execution_loop`), executa apenas a primeira tarefa e para. A causa raiz é um sinal de controle de fluxo (`escalate=True`), emitido por um loop de revisão de código aninhado (`code_review_loop`), que se propaga para o loop pai e causa sua terminação. O objetivo é encontrar uma maneira de conter esse sinal de controle para que ele afete apenas o loop interno.

---

## 2. Contexto Arquitetural

O sistema é um agente multi-estágio que recebe uma solicitação de feature e a transforma em código Flutter. O fluxo principal de execução de código é gerenciado por um `LoopAgent` chamado `task_execution_loop`. A intenção é que este loop itere sobre cada tarefa de um plano de implementação gerado anteriormente.

Dentro de cada iteração deste loop principal, uma sequência de agentes é executada. Um desses agentes é, ele mesmo, outro `LoopAgent` chamado `code_review_loop`, projetado para gerenciar um ciclo de "gerar código -> revisar -> refinar".

### Arquivos e Agentes Envolvidos

*   **Arquivo Principal:** `app/agent.py`
*   **Agentes-Chave:**
    *   `task_execution_loop` (`LoopAgent`): O loop **externo/pai**. Sua função é iterar sobre a lista de tarefas (`implementation_tasks`).
    *   `code_review_loop` (`LoopAgent`): O loop **interno/aninhado**. Sua função é repetir o processo de revisão e refinamento de código até 3 vezes ou até que o código seja aprovado.
    *   `EscalationChecker` (`BaseAgent`): Um agente customizado dentro do `code_review_loop` que verifica o resultado da revisão. Se a nota for `"pass"`, ele emite um evento com `actions.escalate=True` para parar o `code_review_loop` e prosseguir.

---

## 3. Diagnóstico Detalhado do Fluxo de Falha

### 3.1. A Estrutura de Código Problemática

A falha origina-se da forma como os `LoopAgent`s estão aninhados em `app/agent.py`. A estrutura (simplificada) é a seguinte:

```python
# app/agent.py

# Loop Externo (Pai)
task_execution_loop = LoopAgent(
    name="task_execution_loop",
    max_iterations=20,
    sub_agents=[
        TaskManager(name="task_manager"),
        code_generator,
        
        # Loop Interno (Aninhado)
        LoopAgent(
            name="code_review_loop",
            max_iterations=3,
            sub_agents=[
                code_reviewer,
                # Este agente causa o problema
                EscalationChecker(
                    name="code_escalation_checker",
                    review_key="code_review_result",
                ),
                code_refiner,
            ],
            # ...
        ),
        
        # Estes agentes nunca são alcançados após a primeira iteração
        code_approver,
        TaskIncrementer(name="task_incrementer"),
        TaskCompletionChecker(name="task_completion_checker"),
    ],
    after_agent_callback=task_execution_failure_handler,
)
```

### 3.2. O Mecanismo de `escalate`

O agente `EscalationChecker` é projetado para parar seu loop pai (`code_review_loop`) quando o código é aprovado.

```python
# app/agent.py

class EscalationChecker(BaseAgent):
    """Checks evaluation and escalates to stop the loop if grade is 'pass'."""

    def __init__(self, name: str, review_key: str):
        super().__init__(name=name)
        self._review_key = review_key

    async def _run_async_impl(
        self, ctx: InvocationContext
    ) -> AsyncGenerator[Event, None]:
        evaluation_result = ctx.session.state.get(self._review_key)
        if evaluation_result and evaluation_result.get("grade") == "pass":
            logging.info(
                f"[{self.name}] Review for '{self._review_key}' passed."
                " Escalating to stop loop."
            )
            # Emite o sinal que causa a falha
            yield Event(author=self.name, actions=EventActions(escalate=True))
        else:
            # ...
            yield Event(author=self.name)
```

### 3.3. Propagação do Sinal e Falha

1.  Na primeira iteração do `task_execution_loop`, a primeira tarefa é processada.
2.  O `code_review_loop` é iniciado.
3.  O código é gerado e o `code_reviewer` o aprova com `"grade": "pass"`.
4.  O `EscalationChecker` detecta o `"pass"` e corretamente emite o evento com `escalate=True`.
5.  **O `code_review_loop` (interno) captura este sinal e termina sua execução, como esperado.**
6.  **O `task_execution_loop` (externo) também captura este mesmo evento `escalate=True` vindo de seu sub-agente e, seguindo as regras da ADK, termina sua própria execução.**
7.  Como resultado, os agentes `code_approver` e `TaskIncrementer` nunca são executados. O loop principal para com o `current_task_index` ainda em `0`.
8.  O callback `task_execution_failure_handler` é acionado, compara o índice `0` com o total de tarefas (ex: 20), e corretamente reporta a falha.

---

## 4. Evidências do Log

Os logs da última execução confirmam exatamente este fluxo:

1.  **O `EscalationChecker` anuncia sua intenção:**
    ```
    2025-08-08 08:39:49,770 - INFO - agent.py:176 - [code_escalation_checker] Review for 'code_review_result' passed. Escalating to stop loop.
    ```

2.  **O evento com `escalate=True` é gerado:**
    ```json
    {
      "invocationId": "",
      "author": "code_escalation_checker",
      "actions": {
        "escalate": true
      },
      "id": "lyj2YK1F",
      "timestamp": 1754653189.770159
    }
    ```

3.  **Imediatamente a seguir, o callback de falha do loop EXTERNO é chamado, indicando sua terminação:**
    ```json
    {
      "invocationId": "e-7b9e1ad6-4871-43cc-a64e-dce402941fad",
      "author": "task_execution_loop",
      "actions": {
        "stateDelta": {
          "task_execution_failed": true,
          "task_execution_failure_reason": "Limite de tentativas atingido antes de completar todas as tarefas."
        }
      },
      "id": "DWmmKxTr",
      "timestamp": 1754653189.771103
    }
    ```

4.  **A mensagem final confirma a falha:**
    ```
    Processing (FeatureOrchestrator)
    ⚠️ Falha na Execução: Limite de tentativas atingido antes de completar todas as tarefas.
    ```

---

## 5. Tentativa de Correção Anterior (Falha)

Uma tentativa de correção foi feita envolvendo a estrutura do `task_execution_loop` para usar um `SequentialAgent` para agrupar os sub-agentes.

```python
# Tentativa de correção
single_task_pipeline = SequentialAgent(...)
task_execution_loop = LoopAgent(sub_agents=[single_task_pipeline, TaskCompletionChecker(...)])
```

Esta abordagem falhou porque o `SequentialAgent` também propaga o sinal `escalate`. Ao receber o `escalate` de seu filho (`code_review_loop`), ele para sua própria execução e retransmite o sinal para seu pai (`task_execution_loop`), resultando no mesmo comportamento de falha.

---

## 6. Pergunta Central a ser Resolvida

**Como podemos reestruturar o `task_execution_loop` ou seus sub-agentes em `app/agent.py` para que o sinal `escalate=True` emitido pelo `code_review_loop` seja "consumido" ou contido, permitindo que ele finalize apenas o loop de revisão interno, enquanto o `task_execution_loop` principal prossegue para a sua próxima iteração ou para o próximo agente em sua sequência (`code_approver`, `TaskIncrementer`)?**
