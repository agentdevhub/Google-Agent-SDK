# 回调类型

该框架提供了多种回调类型，它们会在智能体执行的不同阶段触发。理解每种回调的触发时机及其接收的上下文信息，是高效使用它们的关键。

## 智能体生命周期回调

这些回调适用于*任何*继承自 `BaseAgent` 的智能体（包括 `LlmAgent`、`SequentialAgent`、`ParallelAgent`、`LoopAgent` 等）。

### 智能体执行前回调

**触发时机：** 在智能体的 `_run_async_impl`（或 `_run_live_impl`）方法执行*之前立即*调用。该回调在智能体的 `InvocationContext` 创建完成之后运行，但*早于*其核心逻辑开始执行。

**用途：** 适用于以下场景：
- 设置仅本次智能体运行所需的资源或状态
- 在执行开始前对会话状态（callback_context.state）进行验证检查
- 记录智能体活动的入口点
- 在核心逻辑使用前修改调用上下文

??? "代码示例"

    ```py
    --8<-- "examples/python/snippets/callbacks/before_agent_callback.py"
    ```

### 智能体执行后回调

**触发时机：** 在智能体的 `_run_async_impl`（或 `_run_live_impl`）方法*成功完成后立即*调用。如果智能体因 `before_agent_callback` 返回内容而被跳过，或是在运行期间设置了 `end_invocation`，则不会触发此回调。

**用途：** 适用于以下场景：
- 执行清理任务
- 进行事后验证
- 记录智能体活动的完成状态
- 修改最终状态
- 增强/替换智能体的最终输出

??? "代码示例"

    ```py
    --8<-- "examples/python/snippets/callbacks/after_agent_callback.py"
    ```

## 大模型交互回调

这些回调专属于 `LlmAgent`，提供了与大模型交互过程中的钩子函数。

### 模型调用前回调

**触发时机：** 在 `generate_content_async`（或等效请求）被发送到大模型之前，`LlmAgent` 流程中触发。

**用途：** 允许检查和修改发送给大模型的请求。典型应用场景包括：
- 添加动态指令
- 根据状态注入少量示例
- 修改模型配置
- 实现防护机制（如敏感词过滤）
- 实现请求级缓存

**返回值影响：**  
若回调返回 `None`，则大模型继续正常流程。若返回 `LlmResponse` 对象，则*跳过*对大模型的调用，直接使用返回的 `LlmResponse` 作为模型响应。此机制可用于实现防护或缓存功能。

??? "代码示例"

    ```py
    --8<-- "examples/python/snippets/callbacks/before_model_callback.py"
    ```

### 模型调用后回调

**触发时机：** 在接收到大模型响应（`LlmResponse`）之后、调用智能体进一步处理之前触发。

**用途：** 允许检查或修改原始的大模型响应。典型应用场景包括：
- 记录模型输出
- 重新格式化响应
- 过滤模型生成的敏感信息
- 从响应中解析结构化数据并存储到 `callback_context.state`
- 处理特定的错误码

??? "代码示例"

    ```py
    --8<-- "examples/python/snippets/callbacks/after_model_callback.py"
    ```

## 工具执行回调

这些回调同样专属于 `LlmAgent`，围绕工具执行过程触发（包括 `FunctionTool`、`AgentTool` 等大模型可能请求的工具）。

### 工具执行前回调

**触发时机：** 在特定工具的 `run_async` 方法被调用前触发，此时大模型已生成对应的函数调用。

**用途：** 允许以下操作：
- 检查并修改工具参数
- 执行前进行授权检查
- 记录工具使用尝试
- 实现工具级缓存

**返回值影响：**
1. 若返回 `None`，则使用（可能被修改过的）`args` 执行工具的 `run_async` 方法  
2. 若返回字典，则*跳过*工具的 `run_async` 方法，直接使用返回字典作为工具调用结果。适用于缓存或覆盖工具行为的场景  

??? "代码示例"

    ```py
    --8<-- "examples/python/snippets/callbacks/before_tool_callback.py"
    ```

### 工具执行后回调

**触发时机：** 在工具的 `run_async` 方法成功完成后立即触发。

**用途：** 允许在工具结果返回给大模型前（可能在摘要处理后）进行检查和修改。适用于：
- 记录工具结果
- 对结果进行后处理或格式化
- 将特定结果保存到会话状态

**返回值影响：**
1. 若返回 `None`，则使用原始 `tool_response`  
2. 若返回新字典，则*替换*原始 `tool_response`，从而修改大模型接收到的结果  

??? "代码示例"

    ```py
    --8<-- "examples/python/snippets/callbacks/after_tool_callback.py"
    ```