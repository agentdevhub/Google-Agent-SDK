!!! warning "高级概念"

    通过直接实现 `_run_async_impl` 来构建自定义智能体虽然能提供强大的控制能力，但比使用预定义的 `LlmAgent` 或标准 `WorkflowAgent` 类型更为复杂。建议先掌握基础智能体类型，再着手处理自定义编排逻辑。

# 自定义智能体

自定义智能体为ADK提供了终极灵活性，允许您通过直接继承 `BaseAgent` 并实现自己的控制流来定义**任意编排逻辑**。这超越了 `SequentialAgent`、`LoopAgent` 和 `ParallelAgent` 的预定义模式，使您能够构建高度定制化的复杂智能体工作流。

## 引言：超越预定义工作流

### 什么是自定义智能体？

自定义智能体本质上是任何继承自 `google.adk.agents.BaseAgent` 并在异步方法 `_run_async_impl` 中实现核心执行逻辑的类。您可以完全控制该方法如何调用其他智能体（子智能体）、管理状态和处理事件。

### 使用场景

虽然标准[工作流智能体](workflow-agents/index.md)（`SequentialAgent`、`LoopAgent`、`ParallelAgent`）涵盖了常见编排模式，但在以下场景需要自定义智能体：

* **条件逻辑**：根据运行时条件或上一步结果执行不同的子智能体或路径
* **复杂状态管理**：实现超越简单顺序传递的精细状态维护逻辑
* **外部集成**：在编排流控制中直接调用外部API、数据库或自定义Python库
* **动态智能体选择**：根据对情境或输入的动态评估选择后续子智能体
* **独特工作流模式**：实现不符合标准顺序、并行或循环结构的编排逻辑

![intro_components.png](../assets/custom-agent-flow.png)

## 实现自定义逻辑

自定义智能体的核心是 `_run_async_impl` 方法，这里定义了其独特行为：

* **签名**：`async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:`
* **异步生成器**：必须是 `async def` 函数并返回 `AsyncGenerator`，以便将子智能体或自身逻辑产生的事件 `yield` 回传给运行器
* **`ctx`（调用上下文）**：提供关键运行时信息访问，最重要的是 `ctx.session.state`——这是自定义智能体编排的步骤间共享数据的主要方式

**`_run_async_impl` 中的关键能力**：

1. **调用子智能体**：使用子智能体的 `run_async` 方法调用它们（通常存储为实例属性如 `self.my_llm_agent`）并生成其事件：
    ```python
    async for event in self.some_sub_agent.run_async(ctx):
        # 可选检查或记录事件
        yield event # 向上传递事件
    ```

2. **状态管理**：通过会话状态字典（`ctx.session.state`）在子智能体调用间传递数据或做出决策：
    ```python
    # 读取前序智能体设置的数据
    previous_result = ctx.session.state.get("some_key")

    # 基于状态决策
    if previous_result == "some_value":
        # ... 调用特定子智能体 ...
    else:
        # ... 调用其他子智能体 ...

    # 存储结果供后续步骤使用（通常通过子智能体的output_key实现）
    # ctx.session.state["my_custom_result"] = "calculated_value"
    ```

3. **实现控制流**：使用标准Python结构（`if`/`elif`/`else`、`for`/`while` 循环、`try`/`except`）创建涉及子智能体的复杂条件或迭代工作流

## 管理子智能体与状态

通常自定义智能体会编排其他智能体（如 `LlmAgent`、`LoopAgent` 等）：

* **初始化**：通常将这些子智能体实例传入自定义智能体的 `__init__` 方法并存储为实例属性（如 `self.story_generator = story_generator_instance`），以便在 `_run_async_impl` 中访问
* **`sub_agents` 列表**：使用 `super().__init__(...)` 初始化 `BaseAgent` 时应传递 `sub_agents` 列表，告知ADK框架该自定义智能体直接管理的子智能体层级关系。这对生命周期管理、内省等框架功能至关重要，即使 `_run_async_impl` 通过 `self.xxx_agent` 直接调用智能体
* **状态**：如前所述，`ctx.session.state` 是子智能体（特别是使用 `output_key` 的 `LlmAgent`）与编排器通信的标准方式，也是编排器向下传递必要输入的通道

## 设计模式示例：`StoryFlowAgent`

通过示例展示自定义智能体的强大能力：包含条件逻辑的多阶段内容生成工作流。

**目标**：创建生成故事→通过评审修订迭代优化→执行最终检查→*若最终语气检查失败则重新生成故事*的系统

**为何自定义**：核心需求是**基于语气检查结果的条件性重新生成**。标准工作流智能体没有基于子智能体任务结果的条件分支功能，需要在编排器中实现自定义Python逻辑（`if tone == "negative": ...`）

---

### 第一部分：简化自定义智能体初始化

定义继承自 `BaseAgent` 的 `StoryFlowAgent`。在 `__init__` 中存储必要子智能体（传入）为实例属性，并告知 `BaseAgent` 框架该自定义智能体直接管理的顶层智能体。

```python
--8<-- "examples/python/snippets/agents/custom-agent/storyflow_agent.py:init"
```

---

### 第二部分：定义自定义执行逻辑

该方法使用标准Python async/await和控制流编排子智能体：

```python
--8<-- "examples/python/snippets/agents/custom-agent/storyflow_agent.py:executionlogic"
```

**逻辑说明**：

1. 初始运行 `story_generator`，其输出预期存入 `ctx.session.state["current_story"]`
2. 运行 `loop_agent`（内部调用 `critic` 和 `reviser` 进行 `max_iterations` 次迭代），它们从/向状态读写 `current_story` 和 `criticism`
3. 运行 `sequential_agent`（调用 `grammar_check` 和 `tone_check`），读取 `current_story` 并向状态写入 `grammar_suggestions` 和 `tone_check_result`
4. **自定义部分**：`if` 语句检查状态中的 `tone_check_result`，若为"negative"则再次调用 `story_generator` 覆盖状态中的 `current_story`，否则结束流程

---

### 第三部分：定义LLM子智能体

这些是标准 `LlmAgent` 定义，负责特定任务。其 `output_key` 参数对将结果存入 `session.state`（供其他智能体或自定义编排器访问）至关重要。

```python
GEMINI_FLASH = "gemini-2.0-flash" # Define model constant
--8<-- "examples/python/snippets/agents/custom-agent/storyflow_agent.py:llmagents"
```

---

### 第四部分：实例化并运行自定义智能体

最后实例化 `StoryFlowAgent` 并照常使用 `Runner`：

```python
--8<-- "examples/python/snippets/agents/custom-agent/storyflow_agent.py:story_flow_agent"
```

（注：完整可运行代码包括导入和执行逻辑，参见下方链接）

---

## 完整代码示例

???+ "Storyflow智能体"

    ```python
    # StoryFlowAgent示例的完整可运行代码
    --8<-- "examples/python/snippets/agents/custom-agent/storyflow_agent.py"
    ```