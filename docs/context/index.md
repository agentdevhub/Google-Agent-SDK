# 上下文

## 什么是上下文

在智能体开发套件（ADK）中，"上下文"指的是智能体及其工具在执行特定操作时可获取的关键信息集合。你可以将其理解为有效处理当前任务或对话轮次所需的背景知识和资源集合。

智能体通常不仅需要最新用户消息才能良好运作。上下文之所以重要，是因为它能实现：

1. **状态维护**：在多次对话中记住细节（如用户偏好、先前的计算结果、购物车中的商品）。这主要通过**会话状态**来管理。
2. **数据传递**：将在某一步骤（如大模型调用或工具执行）中发现或生成的信息传递给后续步骤。会话状态在此也起关键作用。
3. **服务访问**：与框架能力交互，例如：
    * **文件存储**：保存或加载与会话关联的文件或数据块（如PDF、图片、配置文件）
    * **记忆系统**：从过往交互或用户关联的外部知识源中搜索相关信息
    * **认证系统**：请求并获取工具安全访问外部API所需的凭证
4. **身份识别与追踪**：识别当前运行的智能体（`agent.name`）并唯一标记当前请求-响应周期（`invocation_id`），用于日志记录和调试
5. **工具专用操作**：支持工具内的特殊操作，如请求认证或搜索记忆，这些操作需要访问当前交互的详细信息

将所有信息整合在一个完整用户请求到最终响应周期（即**调用**）中的核心载体是`InvocationContext`。不过你通常不需要直接创建或管理这个对象。ADK框架会在调用开始时（例如通过`runner.run_async`）创建它，并将相关上下文信息隐式传递给智能体代码、回调和工具。

```python
# Conceptual Pseudocode: How the framework provides context (Internal Logic)

# runner = Runner(agent=my_root_agent, session_service=..., artifact_service=...)
# user_message = types.Content(...)
# session = session_service.get_session(...) # Or create new

# --- Inside runner.run_async(...) ---
# 1. Framework creates the main context for this specific run
# invocation_context = InvocationContext(
#     invocation_id="unique-id-for-this-run",
#     session=session,
#     user_content=user_message,
#     agent=my_root_agent, # The starting agent
#     session_service=session_service,
#     artifact_service=artifact_service,
#     memory_service=memory_service,
#     # ... other necessary fields ...
# )

# 2. Framework calls the agent's run method, passing the context implicitly
#    (The agent's method signature will receive it, e.g., _run_async_impl(self, ctx: InvocationContext))
# await my_root_agent.run_async(invocation_context)
# --- End Internal Logic ---

# As a developer, you work with the context objects provided in method arguments.
```

## 不同类型的上下文

虽然`InvocationContext`作为综合性的内部容器，但ADK提供了针对特定场景优化的专用上下文对象。这确保你在处理当前任务时拥有合适的工具和权限，而不必处处处理完整上下文的复杂性。以下是你会遇到的不同"变体"：

1.  **`InvocationContext`**
    *   **使用场景**：作为`ctx`参数直接接收于智能体核心实现方法中（`_run_async_impl`、`_run_live_impl`）
    *   **用途**：提供访问当前调用*完整*状态的能力，是最全面的上下文对象
    *   **关键内容**：直接访问`session`（包含`state`和`events`）、当前`agent`实例、`invocation_id`、初始`user_content`、对配置服务的引用（`artifact_service`、`memory_service`、`session_service`），以及与实时/流模式相关的字段
    *   **用例**：主要用于智能体核心逻辑需要直接访问整体会话或服务时，不过状态和文件交互通常委托给使用自有上下文的回调/工具。也用于控制调用本身（如设置`ctx.end_invocation = True`）

    ```python
    # 伪代码：接收InvocationContext的智能体实现
    from google.adk.agents import BaseAgent, InvocationContext
    from google.adk.events import Event
    from typing import AsyncGenerator

    class MyAgent(BaseAgent):
        async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
            # 直接访问示例
            agent_name = ctx.agent.name
            session_id = ctx.session.id
            print(f"Agent {agent_name} running in session {session_id} for invocation {ctx.invocation_id}")
            # ... 使用ctx的智能体逻辑 ...
            yield # ... 事件 ...
    ```

2.  **`ReadonlyContext`**
    *   **使用场景**：仅需读取基础信息且禁止修改的场景（如`InstructionProvider`函数）。也是其他上下文的基类
    *   **用途**：提供对基础上下文细节的安全只读视图
    *   **关键内容**：`invocation_id`、`agent_name`以及当前`state`的只读视图

    ```python
    # 伪代码：接收ReadonlyContext的指令提供器
    from google.adk.agents import ReadonlyContext

    def my_instruction_provider(context: ReadonlyContext) -> str:
        # 只读访问示例
        user_tier = context.state.get("user_tier", "standard") # 可读取状态
        # context.state['new_key'] = 'value' # 这通常会引发错误或无效
        return f"Process the request for a {user_tier} user."
    ```

3.  **`CallbackContext`**
    *   **使用场景**：作为`callback_context`传递给智能体生命周期回调（`before_agent_callback`、`after_agent_callback`）和模型交互回调（`before_model_callback`、`after_model_callback`）
    *   **用途**：便于在回调中检查和修改状态、与文件交互以及访问调用详情
    *   **关键能力（在`ReadonlyContext`基础上增加）**：
        *   **可变的`state`属性**：允许读写会话状态。在此处所做的更改（`callback_context.state['key'] = value`）会被框架跟踪并关联到回调后生成的事件
        *   **文件操作方法**：`load_artifact(filename)`和`save_artifact(filename, part)`方法用于与配置的`artifact_service`交互
        *   直接`user_content`访问

    ```python
    # 伪代码：接收CallbackContext的回调
    from google.adk.agents import CallbackContext
    from google.adk.models import LlmRequest
    from google.genai import types
    from typing import Optional

    def my_before_model_cb(callback_context: CallbackContext, request: LlmRequest) -> Optional[types.Content]:
        # 读写状态示例
        call_count = callback_context.state.get("model_calls", 0)
        callback_context.state["model_calls"] = call_count + 1 # 修改状态

        # 可选加载文件
        # config_part = callback_context.load_artifact("model_config.json")
        print(f"Preparing model call #{call_count + 1} for invocation {callback_context.invocation_id}")
        return None # 允许模型调用继续
    ```

4.  **`ToolContext`**
    *   **使用场景**：作为`tool_context`传递给支持`FunctionTool`的函数以及工具执行回调（`before_tool_callback`、`after_tool_callback`）
    *   **用途**：提供`CallbackContext`的所有功能，外加工具执行必需的特殊方法，如处理认证、搜索记忆和列出文件
    *   **关键能力（在`CallbackContext`基础上增加）**：
        *   **认证方法**：`request_credential(auth_config)`触发认证流程，`get_auth_response(auth_config)`检索用户/系统提供的凭证
        *   **文件列表**：`list_artifacts()`发现会话中可用的文件
        *   **记忆搜索**：`search_memory(query)`查询配置的`memory_service`
        *   **`function_call_id`属性**：标识触发此工具执行的LLM特定函数调用，对于正确关联认证请求或响应至关重要
        *   **`actions`属性**：直接访问此步骤的`EventActions`对象，允许工具发出状态变更、认证请求等信号

    ```python
    # 伪代码：接收ToolContext的工具函数
    from google.adk.agents import ToolContext
    from typing import Dict, Any

    # 假设此函数由FunctionTool包装
    def search_external_api(query: str, tool_context: ToolContext) -> Dict[str, Any]:
        api_key = tool_context.state.get("api_key")
        if not api_key:
            # 定义所需认证配置
            # auth_config = AuthConfig(...)
            # tool_context.request_credential(auth_config) # 请求凭证
            # 使用'actions'属性表示已发出认证请求
            # tool_context.actions.requested_auth_configs[tool_context.function_call_id] = auth_config
            return {"status": "Auth Required"}

        # 使用API密钥...
        print(f"Tool executing for query '{query}' using API key. Invocation: {tool_context.invocation_id}")

        # 可选搜索记忆或列出文件
        # relevant_docs = tool_context.search_memory(f"info related to {query}")
        # available_files = tool_context.list_artifacts()

        return {"result": f"Data for {query} fetched."}
    ```

理解这些不同的上下文对象及其使用场景，是有效管理状态、访问服务和控制ADK应用流程的关键。下一节将详细介绍使用这些上下文执行常见任务的方法。

## 使用上下文的常见任务

了解不同上下文对象后，我们重点介绍构建智能体和工具时如何使用它们完成常见任务。

### 访问信息

经常需要读取存储在上下文中的信息。

*   **读取会话状态**：访问先前步骤保存的数据或用户/应用级设置。使用`state`属性的字典式访问。

    ```python
    # 伪代码：在工具函数中
    from google.adk.agents import ToolContext

    def my_tool(tool_context: ToolContext, **kwargs):
        user_pref = tool_context.state.get("user_display_preference", "default_mode")
        api_endpoint = tool_context.state.get("app:api_endpoint") # 读取应用级状态

        if user_pref == "dark_mode":
            # ... 应用深色模式逻辑 ...
            pass
        print(f"Using API endpoint: {api_endpoint}")
        # ... 工具剩余逻辑 ...

    # 伪代码：在回调函数中
    from google.adk.agents import CallbackContext

    def my_callback(callback_context: CallbackContext, **kwargs):
        last_tool_result = callback_context.state.get("temp:last_api_result") # 读取临时状态
        if last_tool_result:
            print(f"Found temporary result from last tool: {last_tool_result}")
        # ... 回调逻辑 ...
    ```

*   **获取当前标识符**：对基于当前操作的日志记录或自定义逻辑很有用。

    ```python
    # 伪代码：任意上下文中（以ToolContext为例）
    from google.adk.agents import ToolContext

    def log_tool_usage(tool_context: ToolContext, **kwargs):
        agent_name = tool_context.agent_name
        inv_id = tool_context.invocation_id
        func_call_id = getattr(tool_context, 'function_call_id', 'N/A') # ToolContext特有

        print(f"Log: Invocation={inv_id}, Agent={agent_name}, FunctionCallID={func_call_id} - Tool Executed.")
    ```

*   **访问初始用户输入**：回溯启动当前调用的消息。

    ```python
    # 伪代码：在回调中
    from google.adk.agents import CallbackContext

    def check_initial_intent(callback_context: CallbackContext, **kwargs):
        initial_text = "N/A"
        if callback_context.user_content and callback_context.user_content.parts:
            initial_text = callback_context.user_content.parts[0].text or "Non-text input"

        print(f"This invocation started with user input: '{initial_text}'")

    # 伪代码：在智能体的_run_async_impl中
    # async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
    #     if ctx.user_content and ctx.user_content.parts:
    #         initial_text = ctx.user_content.parts[0].text
    #         print(f"Agent logic remembering initial query: {initial_text}")
    #     ...
    ```

### 管理会话状态

状态对于记忆和数据流至关重要。使用`CallbackContext`或`ToolContext`修改状态时，变更会被框架自动跟踪和持久化。

*   **工作原理**：写入`callback_context.state['my_key'] = my_value`或`tool_context.state['my_key'] = my_value`会将此变更添加到当前步骤事件关联的`EventActions.state_delta`中。`SessionService`在持久化事件时应用这些增量。
*   **在工具间传递数据**：

    ```python
    # 伪代码：工具1-获取用户ID
    from google.adk.agents import ToolContext
    import uuid

    def get_user_profile(tool_context: ToolContext) -> dict:
        user_id = str(uuid.uuid4()) # 模拟获取ID
        # 将ID保存到状态供下一个工具使用
        tool_context.state["temp:current_user_id"] = user_id
        return {"profile_status": "ID generated"}

    # 伪代码：工具2-使用状态中的用户ID
    def get_user_orders(tool_context: ToolContext) -> dict:
        user_id = tool_context.state.get("temp:current_user_id")
        if not user_id:
            return {"error": "User ID not found in state"}

        print(f"Fetching orders for user ID: {user_id}")
        # ... 使用user_id获取订单的逻辑 ...
        return {"orders": ["order123", "order456"]}
    ```

*   **更新用户偏好**：

    ```python
    # 伪代码：工具或回调识别偏好
    from google.adk.agents import ToolContext # 或CallbackContext

    def set_user_preference(tool_context: ToolContext, preference: str, value: str) -> dict:
        # 对用户级状态使用'user:'前缀（如果使用持久化SessionService）
        state_key = f"user:{preference}"
        tool_context.state[state_key] = value
        print(f"Set user preference '{preference}' to '{value}'")
        return {"status": "Preference updated"}
    ```

*   **状态前缀**：基础状态是会话特定的，但使用持久化`SessionService`实现（如`DatabaseSessionService`或`VertexAiSessionService`）时，`app:`和`user:`等前缀可表示更广的范围（跨会话的应用级或用户级）。`temp:`可标记仅与当前调用相关的数据。

### 处理文件

使用文件处理与会话关联的文件或大数据块。常见用例：处理上传的文档。

*   **文档摘要器示例流程**：

    1.  **引入引用（如在设置工具或回调中）**：将文档的*路径或URI*保存为文件，而非整个内容。

        ```python
        # 伪代码：在回调或初始工具中
        from google.adk.agents import CallbackContext # 或ToolContext
        from google.genai import types

        def save_document_reference(context: CallbackContext, file_path: str) -> None:
            # 假设file_path类似"gs://my-bucket/docs/report.pdf"或"/local/path/to/report.pdf"
            try:
                # 创建包含路径/URI文本的Part
                artifact_part = types.Part(text=file_path)
                version = context.save_artifact("document_to_summarize.txt", artifact_part)
                print(f"Saved document reference '{file_path}' as artifact version {version}")
                # 如有需要，将文件名存储在状态中供其他工具使用
                context.state["temp:doc_artifact_name"] = "document_to_summarize.txt"
            except ValueError as e:
                print(f"Error saving artifact: {e}") # 例如文件服务未配置
            except Exception as e:
                print(f"Unexpected error saving artifact reference: {e}")

        # 使用示例：
        # save_document_reference(callback_context, "gs://my-bucket/docs/report.pdf")
        ```

    2.  **摘要器工具**：加载文件获取路径/URI，使用适当库读取实际文档内容，生成摘要并返回结果。

        ```python
        # 伪代码：在摘要器工具函数中
        from google.adk.agents import ToolContext
        from google.genai import types
        # 假设可使用google.cloud.storage或内置open等库
        # 假设存在'summarize_text'函数
        # from my_summarizer_lib import summarize_text

        def summarize_document_tool(tool_context: ToolContext) -> dict:
            artifact_name = tool_context.state.get("temp:doc_artifact_name")
            if not artifact_name:
                return {"error": "Document artifact name not found in state."}

            try:
                # 1. 加载包含路径/URI的文件part
                artifact_part = tool_context.load_artifact(artifact_name)
                if not artifact_part or not artifact_part.text:
                    return {"error": f"Could not load artifact or artifact has no text path: {artifact_name}"}

                file_path = artifact_part.text
                print(f"Loaded document reference: {file_path}")

                # 2. 读取实际文档内容（ADK上下文之外）
                document_content = ""
                if file_path.startswith("gs://"):
                    # 示例：使用GCS客户端库下载/读取
                    # from google.cloud import storage
                    # client = storage.Client()
                    # blob = storage.Blob.from_string(file_path, client=client)
                    # document_content = blob.download_as_text() # 或根据格式使用bytes
                    pass # 替换为实际GCS读取逻辑
                elif file_path.startswith("/"):
                     # 示例：使用本地文件系统
                     with open(file_path, 'r', encoding='utf-8') as f:
                         document_content = f.read()
                else:
                    return {"error": f"Unsupported file path scheme: {file_path}"}

                # 3. 摘要内容
                if not document_content:
                     return {"error": "Failed to read document content."}

                # summary = summarize_text(document_content) # 调用你的摘要逻辑
                summary = f"Summary of content from {file_path}" # 占位符

                return {"summary": summary}

            except ValueError as e:
                 return {"error": f"Artifact service error: {e}"}
            except FileNotFoundError:
                 return {"error": f"Local file not found: {file_path}"}
            # except Exception as e: # 捕获GCS等的特定异常
            #      return {"error": f"Error reading document {file_path}: {e}"}

        ```

*   **列出文件**：发现可用文件。

    ```python
    # 伪代码：在工具函数中
    from google.adk.agents import ToolContext

    def check_available_docs(tool_context: ToolContext) -> dict:
        try:
            artifact_keys = tool_context.list_artifacts()
            print(f"Available artifacts: {artifact_keys}")
            return {"available_docs": artifact_keys}
        except ValueError as e:
            return {"error": f"Artifact service error: {e}"}
    ```

### 处理工具认证

安全管理工具所需的API密钥或其他凭证。

```python
# Pseudocode: Tool requiring auth
from google.adk.agents import ToolContext
from google.adk.auth import AuthConfig # Assume appropriate AuthConfig is defined

# Define your required auth configuration (e.g., OAuth, API Key)
MY_API_AUTH_CONFIG = AuthConfig(...)
AUTH_STATE_KEY = "user:my_api_credential" # Key to store retrieved credential

def call_secure_api(tool_context: ToolContext, request_data: str) -> dict:
    # 1. Check if credential already exists in state
    credential = tool_context.state.get(AUTH_STATE_KEY)

    if not credential:
        # 2. If not, request it
        print("Credential not found, requesting...")
        try:
            tool_context.request_credential(MY_API_AUTH_CONFIG)
            # The framework handles yielding the event. The tool execution stops here for this turn.
            return {"status": "Authentication required. Please provide credentials."}
        except ValueError as e:
            return {"error": f"Auth error: {e}"} # e.g., function_call_id missing
        except Exception as e:
            return {"error": f"Failed to request credential: {e}"}

    # 3. If credential exists (might be from a previous turn after request)
    #    or if this is a subsequent call after auth flow completed externally
    try:
        # Optionally, re-validate/retrieve if needed, or use directly
        # This might retrieve the credential if the external flow just completed
        auth_credential_obj = tool_context.get_auth_response(MY_API_AUTH_CONFIG)
        api_key = auth_credential_obj.api_key # Or access_token, etc.

        # Store it back in state for future calls within the session
        tool_context.state[AUTH_STATE_KEY] = auth_credential_obj.model_dump() # Persist retrieved credential

        print(f"Using retrieved credential to call API with data: {request_data}")
        # ... Make the actual API call using api_key ...
        api_result = f"API result for {request_data}"

        return {"result": api_result}
    except Exception as e:
        # Handle errors retrieving/using the credential
        print(f"Error using credential: {e}")
        # Maybe clear the state key if credential is invalid?
        # tool_context.state[AUTH_STATE_KEY] = None
        return {"error": "Failed to use credential"}

```
*记住：`request_credential`会暂停工具并发出需要认证的信号。用户/系统提供凭证后，在后续调用中使用`get_auth_response`（或再次检查状态）允许工具继续。*框架隐式使用`tool_context.function_call_id`来关联请求和响应。

### 利用记忆系统

从过往或外部源访问相关信息。

```python
# Pseudocode: Tool using memory search
from google.adk.agents import ToolContext

def find_related_info(tool_context: ToolContext, topic: str) -> dict:
    try:
        search_results = tool_context.search_memory(f"Information about {topic}")
        if search_results.results:
            print(f"Found {len(search_results.results)} memory results for '{topic}'")
            # Process search_results.results (which are SearchMemoryResponseEntry)
            top_result_text = search_results.results[0].text
            return {"memory_snippet": top_result_text}
        else:
            return {"message": "No relevant memories found."}
    except ValueError as e:
        return {"error": f"Memory service error: {e}"} # e.g., Service not configured
    except Exception as e:
        return {"error": f"Unexpected error searching memory: {e}"}
```

### 高级：直接使用`InvocationContext`

虽然大多数交互通过`CallbackContext`或`ToolContext`进行，但有时智能体核心逻辑（`_run_async_impl`/`_run_live_impl`）需要直接访问。

```python
# Pseudocode: Inside agent's _run_async_impl
from google.adk.agents import InvocationContext, BaseAgent
from google.adk.events import Event
from typing import AsyncGenerator

class MyControllingAgent(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
        # Example: Check if a specific service is available
        if not ctx.memory_service:
            print("Memory service is not available for this invocation.")
            # Potentially change agent behavior

        # Example: Early termination based on some condition
        if ctx.session.state.get("critical_error_flag"):
            print("Critical error detected, ending invocation.")
            ctx.end_invocation = True # Signal framework to stop processing
            yield Event(author=self.name, invocation_id=ctx.invocation_id, content="Stopping due to critical error.")
            return # Stop this agent's execution

        # ... Normal agent processing ...
        yield # ... event ...
```

设置`ctx.end_invocation = True`是一种优雅停止整个请求-响应周期的方式，可从智能体或其回调/工具内部（通过各自可访问底层`InvocationContext`标志的上下文对象）执行。

## 关键要点与最佳实践

*   **使用正确的上下文**：始终使用提供的最特定上下文对象（工具/工具回调中用`ToolContext`，智能体/模型回调中用`CallbackContext`，适用时用`ReadonlyContext`）。仅在必要时在`_run_async_impl`/`_run_live_impl`中直接使用完整`InvocationContext`（`ctx`）
*   **状态管理数据流**：`context.state`是在调用内共享数据、记住偏好和管理对话记忆的主要方式。使用持久存储时，应谨慎使用前缀（`app:`、`user:`、`temp:`）
*   **文件处理大对象**：使用`context.save_artifact`和`context.load_artifact`管理文件引用（如路径或URI）或大数据块。存储引用，按需加载内容
*   **跟踪变更**：通过上下文方法对状态或文件所做的修改会自动关联到当前步骤的`EventActions`并由`SessionService`处理
*   **从简单开始**：先关注`state`和基础文件使用。随着需求复杂化，再探索认证、记忆和高级`InvocationContext`字段（如实时流相关字段）

通过理解并有效使用这些上下文对象，你可以用ADK构建更复杂、有状态且能力更强的智能体。