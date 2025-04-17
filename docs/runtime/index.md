# 运行时系统

## 什么是运行时？

ADK Runtime 是支撑智能体应用在用户交互过程中运行的核心引擎。这个系统负责调度您定义的智能体、工具和回调函数，响应用户输入来协调执行流程，管理信息流转、状态变更以及与外部服务（如大模型或存储系统）的交互。

您可以将 Runtime 视为智能体应用的**"发动机"**。您负责定义各个组件（智能体、工具），而 Runtime 则处理它们之间的协作运行机制以满足用户请求。

## 核心概念：事件循环

ADK Runtime 的核心运行机制是**事件循环**。这个循环在 `Runner` 组件与您定义的"执行逻辑"（包括智能体、大模型调用、回调和工具）之间建立双向通信通道。

![intro_components.png](../assets/event-loop.png)

简单来说：

1. `Runner` 接收用户查询后，请求主 `Agent` 开始处理
2. `Agent`（及其关联逻辑）持续运行直到需要报告结果（如生成响应、调用工具请求或状态变更）——此时会**产出**一个 `Event`
3. `Runner` 接收该 `Event` 后，处理相关操作（如通过 `Services` 保存状态变更），并将事件转发（例如到用户界面）
4. 只有当 `Runner` 完成事件处理后，`Agent` 的逻辑才会从暂停处**恢复**执行，此时可能已观察到 Runner 提交的变更效果
5. 该循环持续进行，直到智能体对当前用户查询不再产出新事件

这种事件驱动的循环机制是 ADK 执行智能体代码的基础模式。

## 核心机制：事件循环详解

事件循环定义了 `Runner` 与自定义代码（智能体、工具、回调，在设计文档中统称为"执行逻辑"或"逻辑组件"）之间的交互范式，明确划分了职责边界：

### Runner 的角色（协调中枢）

`Runner` 作为单次用户调用的中央协调器，在循环中承担以下职责：

1. **初始化**：接收终端用户查询（`new_message`），通常通过 `SessionService` 将其追加到会话历史
2. **启动**：调用主智能体的执行方法（如 `agent_to_run.run_async(...)`）启动事件生成流程
3. **接收与处理**：等待智能体逻辑 `yield` 产出 `Event`。收到事件后立即处理：
      * 使用配置的 `Services`（`SessionService`、`ArtifactService`、`MemoryService`）提交 `event.actions` 中的变更（如 `state_delta`、`artifact_delta`）
      * 执行其他内部簿记操作
4. **向上传递**：将处理完成的事件转发（如传递给调用应用或界面渲染）
5. **迭代**：通知智能体逻辑已完成当前事件处理，允许其恢复执行并产出*下一个*事件

*Runner 循环概念示意：*

```py
# Simplified view of Runner's main loop logic
def run(new_query, ...) -> Generator[Event]:
    # 1. Append new_query to session event history (via SessionService)
    session_service.append_event(session, Event(author='user', content=new_query))

    # 2. Kick off event loop by calling the agent
    agent_event_generator = agent_to_run.run_async(context)

    async for event in agent_event_generator:
        # 3. Process the generated event and commit changes
        session_service.append_event(session, event) # Commits state/artifact deltas etc.
        # memory_service.update_memory(...) # If applicable
        # artifact_service might have already been called via context during agent run

        # 4. Yield event for upstream processing (e.g., UI rendering)
        yield event
        # Runner implicitly signals agent generator can continue after yielding
```

### 执行逻辑的角色（智能体、工具、回调）

智能体、工具和回调中的代码负责实际计算和决策制定，其与循环的交互包括：

1. **执行**：基于当前 `InvocationContext`（包括恢复执行时的会话状态）运行逻辑
2. **产出**：当需要通信时（发送消息、调用工具、报告状态变更），构建包含相关内容和操作的 `Event`，然后通过 `yield` 将事件传回 `Runner`
3. **暂停**：关键的是，智能体逻辑在 `yield` 语句后**立即暂停**，等待 `Runner` 完成步骤3（处理与提交）
4. **恢复**：只有当 `Runner` 处理完产出的事件后，智能体逻辑才从 `yield` 后的语句继续执行
5. **观察更新状态**：恢复后，智能体逻辑可以可靠访问反映先前产出事件所提交变更的会话状态（`ctx.session.state`）

*执行逻辑概念示意：*

```py
# Simplified view of logic inside Agent.run_async, callbacks, or tools

# ... previous code runs based on current state ...

# 1. Determine a change or output is needed, construct the event
# Example: Updating state
update_data = {'field_1': 'value_2'}
event_with_state_change = Event(
    author=self.name,
    actions=EventActions(state_delta=update_data),
    content=types.Content(parts=[types.Part(text="State updated.")])
    # ... other event fields ...
)

# 2. Yield the event to the Runner for processing & commit
yield event_with_state_change
# <<<<<<<<<<<< EXECUTION PAUSES HERE >>>>>>>>>>>>

# <<<<<<<<<<<< RUNNER PROCESSES & COMMITS THE EVENT >>>>>>>>>>>>

# 3. Resume execution ONLY after Runner is done processing the above event.
# Now, the state committed by the Runner is reliably reflected.
# Subsequent code can safely assume the change from the yielded event happened.
val = ctx.session.state['field_1']
# here `val` is guaranteed to be "value_2" (assuming Runner committed successfully)
print(f"Resumed execution. Value of field_1 is now: {val}")

# ... subsequent code continues ...
# Maybe yield another event later...
```

这种由 `Event` 对象协调的、`Runner` 与执行逻辑之间的产出/暂停/恢复协作循环，构成了 ADK Runtime 的核心架构。

## 运行时关键组件

ADK Runtime 中多个组件协同工作来执行智能体调用，理解它们的角色有助于厘清事件循环的运作机制：

1. ### `Runner`

      * **角色**：单次用户查询（`run_async`）的主入口和协调器
      * **功能**：管理整体事件循环，接收执行逻辑产出的事件，协调服务处理并提交事件操作（状态/产物变更），将处理完成的事件向上传递（如到UI）。本质上是基于产出事件驱动逐轮对话（定义于 `google.adk.runners.runner.py`）

2. ### 执行逻辑组件

      * **角色**：包含自定义代码和智能体核心能力的部分
      * **组成**：
      * `Agent`（`BaseAgent`、`LlmAgent` 等）：处理信息并决策行动的主逻辑单元，实现产出事件的 `_run_async_impl` 方法
      * `Tools`（`BaseTool`、`FunctionTool`、`AgentTool` 等）：智能体（通常通过 `LlmAgent`）与外界交互或执行特定任务的外部功能，执行后返回结果并包装为事件
      * `Callbacks`（函数）：附加到智能体的用户定义函数（如 `before_agent_callback`、`after_model_callback`），挂钩到执行流特定节点，可能修改行为或状态，其效果通过事件捕获
      * **功能**：执行实际思考、计算或外部交互，通过**产出 `Event` 对象**来通信结果或需求，并暂停直到 Runner 处理完成

3. ### `Event`

      * **角色**：`Runner` 与执行逻辑间传递的消息载体
      * **功能**：表示原子事件（用户输入、智能体文本、工具调用/结果、状态变更请求、控制信号），既携带事件内容也包含预期副作用（`actions` 如 `state_delta`）（定义于 `google.adk.events.event.py`）

4. ### `Services`

      * **角色**：管理持久化或共享资源的后端组件，主要由 `Runner` 在事件处理时使用
      * **组成**：
      * `SessionService`（`BaseSessionService`、`InMemorySessionService` 等）：管理 `Session` 对象，包括保存/加载、对会话状态应用 `state_delta`、将事件追加到 `event history`
      * `ArtifactService`（`BaseArtifactService`、`InMemoryArtifactService`、`GcsArtifactService` 等）：管理二进制产物数据的存储检索。虽然 `save_artifact` 通过上下文在执行逻辑中调用，但事件中的 `artifact_delta` 向Runner/SessionService确认操作
      * `MemoryService`（`BaseMemoryService` 等）：（可选）管理用户跨会话的长期语义记忆
      * **功能**：提供持久化层。`Runner` 与之交互以确保 `event.actions` 信号的变化在*执行逻辑恢复前*可靠存储

5. ### `Session`

      * **角色**：存储用户与应用间*特定对话*状态与历史的数据容器
      * **功能**：存储当前 `state` 字典、所有历史 `events`（`event history`）列表及相关产物引用，是由 `SessionService` 管理的主要交互记录（定义于 `google.adk.sessions.session.py`）

6. ### `Invocation`

      * **角色**：表示从 `Runner` 接收*单次*用户查询到智能体逻辑停止产出事件期间所有发生的概念术语
      * **功能**：一次调用可能涉及多次智能体运行（如使用智能体转移或 `AgentTool`）、多次大模型调用、工具执行和回调执行，全部通过 `InvocationContext` 中的单个 `invocation_id` 关联

这些参与者通过事件循环持续交互来处理用户请求。

## 运作机制：简化调用流程

以下展示涉及大模型智能体调用工具的典型用户查询简化流程：

![intro_components.png](../assets/invocation-flow.png)

### 逐步解析

1. **用户输入**：用户发送查询（如"法国首都是哪里？"）
2. **Runner启动**：`Runner.run_async` 开始运行，与 `SessionService` 交互加载相关 `Session`，将用户查询作为首个 `Event` 加入会话历史，准备 `InvocationContext`（`ctx`）
3. **智能体执行**：`Runner` 调用指定根智能体（如 `LlmAgent`）的 `agent.run_async(ctx)`
4. **大模型调用（示例）**：`Agent_Llm` 判断需要信息（可能通过调用工具），准备 `LLM` 请求。假设大模型决定调用 `MyTool`
5. **产出函数调用事件**：`Agent_Llm` 接收大模型的 `FunctionCall` 响应，包装为 `Event(author='Agent_Llm', content=Content(parts=[Part(function_call=...)]))` 后 `yield` 该事件
6. **智能体暂停**：`Agent_Llm` 在 `yield` 后立即暂停执行
7. **Runner处理**：`Runner` 接收函数调用事件，传递给 `SessionService` 记录历史，`Runner` 将事件向上传递给 `User`（或应用）
8. **智能体恢复**：`Runner` 通知事件处理完成，`Agent_Llm` 恢复执行
9. **工具执行**：`Agent_Llm` 内部流程继续执行请求的 `MyTool`，调用 `tool.run_async(...)`
10. **工具返回结果**：`MyTool` 执行并返回结果（如 `{'result': 'Paris'}`）
11. **产出函数响应事件**：智能体（`Agent_Llm`）将工具结果包装为包含 `FunctionResponse`（如 `Event(author='Agent_Llm', content=Content(role='user', parts=[Part(function_response=...)]))`）的 `Event`。若工具修改状态（`state_delta`）或保存产物（`artifact_delta`），事件可能还包含 `actions`。智能体 `yield` 该事件
12. **智能体暂停**：`Agent_Llm` 再次暂停
13. **Runner处理**：`Runner` 接收函数响应事件，传递给 `SessionService` 应用 `state_delta`/`artifact_delta` 并加入历史，`Runner` 向上传递事件
14. **智能体恢复**：`Agent_Llm` 恢复执行，此时工具结果和状态变更已确认提交
15. **最终大模型调用（示例）**：`Agent_Llm` 将工具结果返回 `LLM` 生成自然语言响应
16. **产出最终文本事件**：`Agent_Llm` 接收大模型的最终文本，包装为 `Event(author='Agent_Llm', content=Content(parts=[Part(text=...)]))` 后 `yield` 产出
17. **智能体暂停**：`Agent_Llm` 暂停
18. **Runner处理**：`Runner` 接收最终文本事件，传递给 `SessionService` 记录历史，向上传递给 `User`，通常标记为 `is_final_response()`
19. **智能体恢复并结束**：`Agent_Llm` 恢复，完成本次调用任务后其 `run_async` 生成器终止
20. **Runner完成**：`Runner` 检测智能体生成器耗尽，结束本次调用的循环

这种产出/暂停/处理/恢复的循环机制确保状态变更被一致应用，且执行逻辑在产出事件后总是基于最新提交的状态运行。

## 重要运行时行为

理解 ADK Runtime 处理状态、流式输出和异步操作的关键特性，对于构建稳定高效的智能体至关重要。

### 状态更新与提交时机

* **规则**：当代码（在智能体、工具或回调中）修改会话状态（如 `context.state['my_key'] = 'new_value'`）时，变更最初记录在当前 `InvocationContext` 的本地副本中。只有当中包含对应 `state_delta` 的 `Event` 被代码 `yield` 产出且经 `Runner` 处理后，变更才**保证被持久化**（由 `SessionService` 保存）

* **影响**：从 `yield` 恢复后运行的代码可以可靠假定*产出事件*中的状态变更已提交

```py
# Inside agent logic (conceptual)

# 1. Modify state
ctx.session.state['status'] = 'processing'
event1 = Event(..., actions=EventActions(state_delta={'status': 'processing'}))

# 2. Yield event with the delta
yield event1
# --- PAUSE --- Runner processes event1, SessionService commits 'status' = 'processing' ---

# 3. Resume execution
# Now it's safe to rely on the committed state
current_status = ctx.session.state['status'] # Guaranteed to be 'processing'
print(f"Status after resuming: {current_status}")
```

### 会话状态的"脏读"

* **定义**：虽然提交发生在产出*之后*，但在*同次调用内*后续运行（且*状态变更事件尚未实际产出和处理前*）的代码**常可观察到本地未提交的变更**，这种现象称为"脏读"
* **示例**：

```py
# Code in before_agent_callback
callback_context.state['field_1'] = 'value_1'
# State is locally set to 'value_1', but not yet committed by Runner

# ... agent runs ...

# Code in a tool called later *within the same invocation*
# Readable (dirty read), but 'value_1' isn't guaranteed persistent yet.
val = tool_context.state['field_1'] # 'val' will likely be 'value_1' here
print(f"Dirty read value in tool: {val}")

# Assume the event carrying the state_delta={'field_1': 'value_1'}
# is yielded *after* this tool runs and is processed by the Runner.
```

* **影响**：
  * **优势**：允许复杂步骤中不同逻辑部分（如多次回调或工具调用）无需等待完整产出/提交周期即可通过状态协调
  * **注意**：关键逻辑过度依赖脏读存在风险。若调用在携带 `state_delta` 的事件被 `Runner` 处理前失败，未提交的变更将丢失。关键状态转换应确保关联事件能被成功处理

### 流式与非流式输出（`partial=True`）

这主要涉及大模型响应处理方式，特别是使用流式生成API时：

* **流式**：大模型逐令牌或分块生成响应
  * 框架（通常在 `BaseLlmFlow` 内）为单个概念响应产出多个 `Event` 对象，多数事件带 `partial=True` 标记
  * `Runner` 收到含 `partial=True` 标记的事件时通常**立即向上传递**（供UI显示）但**跳过处理其 `actions`**（如 `state_delta`）
  * 最终框架会为该响应产出标记为非分块（`partial=False` 或通过 `turn_complete=True` 隐式标记）的结束事件
  * `Runner` **仅完整处理该结束事件**，提交关联的 `state_delta` 或 `artifact_delta`
* **非流式**：大模型一次性生成完整响应。框架产出单个非分块标记事件，`Runner` 完整处理
* **意义**：确保基于大模型*完整*响应原子性应用状态变更，同时允许UI在生成过程中渐进显示文本

## 异步优先（`run_async`）

* **核心设计**：ADK Runtime 基于 Python 的 `asyncio` 库构建，高效处理并发操作（如等待大模型响应或工具执行）而不阻塞
* **主入口**：`Runner.run_async` 是执行智能体调用的主要方法，所有核心可运行组件（智能体、特定流程）内部均使用 `async def` 方法
* **同步便捷接口（`run`）**：同步方法 `Runner.run` 主要为便捷性存在（如简单脚本或测试环境），内部通常只是调用 `Runner.run_async` 并管理异步事件循环执行
* **开发体验**：建议应用逻辑（如使用 ADK 的 web 服务器）采用 `asyncio` 设计
* **同步回调/工具**：框架可无缝处理 `async def` 和常规 `def` 函数形式的工具或回调。长时间运行的*同步*工具或回调（特别是阻塞I/O操作）可能阻塞主 `asyncio` 事件循环。框架可能通过 `asyncio.to_thread` 等机制在独立线程池运行此类阻塞代码以避免影响其他异步任务。但CPU密集型同步代码仍会阻塞所在线程

理解这些行为有助于编写更健壮的 ADK 应用，并调试与状态一致性、流式更新和异步执行相关的问题。