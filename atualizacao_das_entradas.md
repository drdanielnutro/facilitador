### **Plano de Implementação: Agente `DocumentLoader` Autônomo**

#### **Visão Geral e Objetivo**

O objetivo desta implementação é refatorar o sistema para que ele carregue seus documentos de contexto (`.md` do diretório `docs/`) de forma autônoma, diretamente do sistema de arquivos. Isso elimina a dependência atual da inserção manual de documentos no prompt do usuário, tornando o agente mais robusto, eficiente e verdadeiramente autônomo.

Substituiremos a lógica de extração de documentos do `input_processor` por um novo agente especializado, o `DocumentLoader`, que será o primeiro passo no pipeline de processamento de contexto.

---

### **Plano de Implementação Detalhado**

#### **Fase 1: Refatorar o `input_processor` e Introduzir o `DocumentLoader`**

Nesta fase, realizaremos a principal modificação no código. Substituiremos a lógica antiga de extração de documentos por uma mais simples e introduziremos a classe `DocumentLoader`.

*   **Arquivo a ser Editado:** `/Users/institutorecriare/VSCodeProjects/facilitador/facilitador/app/agent.py`
*   **Ação:** Substituição de Código

##### **Trecho de Código a ser REMOVIDO:**

```python
def unpack_extracted_input_callback(callback_context: CallbackContext) -> None:
    """Unpacks the extracted_input dictionary into the session state."""
    if "extracted_input" in callback_context.state:
        extracted_input_str = callback_context.state["extracted_input"]
        try:
            if isinstance(extracted_input_str, str):
                if "```json" in extracted_input_str:
                    extracted_input_str = (
                        extracted_input_str.split("```json")[1]
                        .split("```")[0]
                        .strip()
                    )
                extracted_input = json.loads(extracted_input_str)
            elif isinstance(extracted_input_str, dict):
                extracted_input = extracted_input_str
            else:
                return

            if isinstance(extracted_input, dict):
                for key, value in extracted_input.items():
                    callback_context.state[key] = value
                
                # Store original documents in a separate structure for selective access
                docs = {}
                if "especificacao_tecnica_da_ui" in callback_context.state:
                    docs["ui_spec"] = callback_context.state.get("especificacao_tecnica_da_ui", "")
                if "contexto_api" in callback_context.state:
                    docs["api_context"] = callback_context.state.get("contexto_api", "")
                if "fonte_da_verdade_ux" in callback_context.state:
                    docs["ux_truth"] = callback_context.state.get("fonte_da_verdade_ux", "")
                
                if docs:
                    callback_context.state["original_docs"] = docs
                    logging.info(f"Stored original docs with keys: {list(docs.keys())}")
        except (json.JSONDecodeError, IndexError):
            pass
```

##### **Trecho de Código a ser INCORPORADO (Substituição):**

```python
def unpack_extracted_input_callback(callback_context: CallbackContext) -> None:
    """Unpacks the extracted_input dictionary into the session state."""
    if "extracted_input" in callback_context.state:
        extracted_input_str = callback_context.state["extracted_input"]
        try:
            if isinstance(extracted_input_str, str):
                if "```json" in extracted_input_str:
                    extracted_input_str = (
                        extracted_input_str.split("```json")[1]
                        .split("```")[0]
                        .strip()
                    )
                extracted_input = json.loads(extracted_input_str)
            elif isinstance(extracted_input_str, dict):
                extracted_input = extracted_input_str
            else:
                return

            if isinstance(extracted_input, dict):
                # A lógica de extração de documentos foi removida daqui.
                # A função agora apenas desempacota o input principal.
                for key, value in extracted_input.items():
                    callback_context.state[key] = value
        except (json.JSONDecodeError, IndexError):
            pass
```

---

#### **Fase 1.5: Definição e Implementação do `DocumentLoader`**

Para garantir que os documentos sejam carregados corretamente e que os agentes subsequentes tenham acesso a eles, definimos a classe `DocumentLoader` como um `BaseAgent`. Este agente será responsável por ler os arquivos do diretório `docs/` e popular o estado da sessão com as chaves exatas que a lógica anterior utilizava.

*   **Arquivo a ser Editado:** `/Users/institutorecriare/VSCodeProjects/facilitador/facilitador/app/agent.py`
*   **Ação:** Adição de Nova Classe

##### **Código Completo da Classe `DocumentLoader` a ser ADICIONADO:**

```python
import os
import logging
from google.adk.agents import BaseAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events import Event

# Adicionar esta classe antes da definição do `complete_pipeline`

class DocumentLoader(BaseAgent):
    """
    Um agente programático que carrega documentos de contexto do sistema de arquivos
    para o estado da sessão.
    """

    def __init__(self, name: str):
        super().__init__(name=name)
        # O caminho base para o diretório 'docs'.
        # Em um cenário real, isso poderia vir de uma configuração.
        self._docs_path = "/Users/institutorecriare/VSCodeProjects/facilitador/facilitador/docs"

    async def _run_async_impl(
        self, ctx: InvocationContext
    ) -> AsyncGenerator[Event, None]:
        """
        Executa a lógica de carregamento de documentos.
        """
        logging.info(f"[{self.name}] Iniciando carregamento de documentos de '{self._docs_path}'...")

        # Mapeamento das chaves de estado para os nomes de arquivo e chaves do dicionário.
        # Isso garante que as chaves usadas pelos outros agentes sejam respeitadas.
        doc_map = {
            "especificacao_tecnica_da_ui": {"file": "especificacao_tecnica_da_ui.md", "dict_key": "ui_spec"},
            "contexto_api": {"file": "contexto_api.md", "dict_key": "api_context"},
            "fonte_da_verdade_ux": {"file": "fonte_da_verdade.md", "dict_key": "ux_truth"},
        }

        # Dicionário para armazenar os documentos para a chave 'original_docs'.
        original_docs = {}
        
        for state_key, info in doc_map.items():
            file_path = os.path.join(self._docs_path, info["file"])
            try:
                # Simula a leitura de um arquivo. Em uma implementação real,
                # poderia usar uma ferramenta como 'read_file'.
                with open(file_path, "r", encoding="utf-8") as f:
                    content = f.read()

                # 1. Popula a chave de estado direta (ex: 'especificacao_tecnica_da_ui').
                ctx.session.state[state_key] = content
                
                # 2. Popula o dicionário que se tornará 'original_docs'.
                original_docs[info["dict_key"]] = content
                
                logging.info(f"[{self.name}] Documento '{info['file']}' carregado com sucesso para o estado.")

            except FileNotFoundError:
                error_message = f"DOCUMENTO NÃO ENCONTRADO: {info['file']}"
                ctx.session.state[state_key] = error_message
                original_docs[info["dict_key"]] = error_message
                logging.warning(f"[{self.name}] {error_message}")
            except Exception as e:
                error_message = f"ERRO AO LER DOCUMENTO: {info['file']} - {e}"
                ctx.session.state[state_key] = error_message
                original_docs[info["dict_key"]] = error_message
                logging.error(f"[{self.name}] {error_message}")

        # 3. Popula a chave de estado 'original_docs' com o dicionário completo.
        ctx.session.state["original_docs"] = original_docs
        logging.info(f"[{self.name}] Estrutura 'original_docs' populada com as chaves: {list(original_docs.keys())}")

        # Sinaliza a conclusão da sua tarefa.
        yield Event(author=self.name)

```

---

#### **Fase 2: Integrar o `DocumentLoader` no Pipeline Principal**

Agora, com o `DocumentLoader` definido, vamos inseri-lo no pipeline `complete_pipeline`. Ele será executado logo após o `input_processor`, garantindo que os documentos estejam disponíveis no estado antes que o `planning_pipeline` comece.

*   **Arquivo a ser Editado:** `/Users/institutorecriare/VSCodeProjects/facilitador/facilitador/app/agent.py`
*   **Ação:** Substituição de Código

##### **Trecho de Código a ser REMOVIDO:**

```python
# --- COMPLETE PIPELINE WITH INPUT PROCESSING ---

complete_pipeline = SequentialAgent(
    name="complete_pipeline",
    description="Complete pipeline from input processing to code generation",
    sub_agents=[
        input_processor,        # Primeiro processa o input
        planning_pipeline,      # Depois planeja
        execution_pipeline      # Por fim executa
    ]
)
```

##### **Trech_de_Código a ser INCORPORADO (Substituição):**

```python
# --- COMPLETE PIPELINE WITH INPUT PROCESSING ---

complete_pipeline = SequentialAgent(
    name="complete_pipeline",
    description="Complete pipeline from input processing to code generation",
    sub_agents=[
        input_processor,        # 1. Extrai o pedido da feature do prompt do usuário
        DocumentLoader(name="document_loader"), # 2. Carrega docs de contexto do disco
        planning_pipeline,      # 3. Planeja a implementação
        execution_pipeline      # 4. Executa o plano e gera o código
    ]
)
```

---

### **Análise de Riscos e Mitigações (Incorporado no Design)**

1.  **Risco:** O diretório `docs/` não existe ou está no lugar errado.
    *   **Mitigação:** O `DocumentLoader` usa um `try...except FileNotFoundError`. Se um arquivo não for encontrado, ele não quebrará o pipeline. Em vez disso, registrará um aviso e colocará uma mensagem "DOCUMENTO NÃO ENCONTRADO" no estado da sessão. Os agentes seguintes verão essa mensagem em vez do conteúdo do documento.

2.  **Risco:** Um arquivo de documentação está corrompido ou tem problemas de codificação.
    *   **Mitigação:** O `DocumentLoader` inclui um bloco `except Exception as e` genérico para capturar quaisquer outros erros de I/O. Ele registrará o erro e, novamente, colocará uma mensagem de erro no estado, garantindo a resiliência do pipeline.

3.  **Risco:** O `input_processor` simplificado falha em extrair o `feature_snippet`.
    *   **Mitigação:** O `FeatureOrchestrator` já possui uma verificação para `if "feature_snippet" not in ctx.session.state`, que lidará com esse caso de forma elegante, solicitando ao usuário que forneça uma descrição clara. Essa lógica de fallback existente continua a funcionar perfeitamente.

---

### **Plano de Verificação e Testes**

Após aplicar as mudanças, siga estes passos para garantir que tudo funciona como esperado:

1.  **Teste de Carga do Módulo:** Execute um teste simples para garantir que não há erros de sintaxe no arquivo.
    ```bash
    python -c "from app.agent import root_agent; print('Módulo app/agent.py carregado com sucesso!')"
    ```

2.  **Teste de Execução do Pipeline:** Execute o agente com um prompt simples que não contenha nenhuma tag de documento.
    ```bash
    # Exemplo de prompt para run_agent.py
    "Implemente um botão de login simples na tela inicial."
    ```

3.  **Verificação dos Logs:** Observe a saída do console. Você deve ver as seguintes mensagens de log, confirmando que o `DocumentLoader` executou com sucesso:
    ```
    INFO: [document_loader] Loading documentation from 'docs'...
    INFO: Successfully loaded 'especificacao_tecnica_da_ui.md' into state.
    INFO: Successfully loaded 'contexto_api.md' into state.
    INFO: Successfully loaded 'fonte_da_verdade.md' into state.
    ```

4.  **Verificação da Qualidade:** O resultado final (código gerado) deve ser de alta qualidade, pois os agentes `code_generator` e `code_reviewer` agora têm acesso ao contexto completo que foi carregado automaticamente.

---
