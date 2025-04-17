# LLM 智能体

`LlmAgent`（常简称为`Agent`）是ADK中的核心组件，作为应用程序的"思考中枢"。它借助大模型（LLM）的强大能力进行推理、理解自然语言、做出决策、生成响应以及与工具交互。

与遵循预定义执行路径的确定性[工作流智能体](workflow-agents/index.md)不同，`LlmAgent`的行为具有非确定性。它利用大模型来解析指令和上下文，动态决定后续操作、选择使用哪些工具（如有需要）或是否将控制权转移给其他智能体。

构建高效的`LlmAgent`需要定义其身份标识，通过指令明确引导其行为，并为其配备必要的工具和能力。

## 定义智能体身份与用途

首先需要明确智能体*是什么*以及*用途*。

* **`name`（必填）：** 每个智能体都需要唯一的字符串标识符。该`name`对内部运作至关重要，特别是在多智能体系统中，智能体需要相互引用或委派任务时。选择能反映智能体功能的描述性名称（例如`customer_support_router`、`billing_inquiry_agent`）。避免使用`user`等保留名称。

* **`description`（可选，推荐用于多智能体系统）：** 提供智能体能力的简明摘要。该描述主要供*其他*大模型智能体判断是否应将任务路由到本智能体。描述需足够具体以区别于同类智能体（例如"处理当前账单查询"，而非简单的"账单智能体"）。

* **`model`（必填）：** 指定支撑该智能体推理的基础大模型。这是一个字符串标识符，如`"gemini-2.0-flash"`。模型选择会影响智能体的能力、成本和性能。可用选项及考量因素请参阅[模型](models.md)页面。

```python
# Example: Defining the basic identity
capital_agent = LlmAgent(
    model="gemini-2.0-flash",
    name="capital_agent",
    description="Answers user questions about the capital city of a given country."
    # instruction and tools will be added next
)
```

## 引导智能体：指令（`instruction`）

`instruction`参数可以说是塑造`LlmAgent`行为最关键的因素。这个字符串（或返回字符串的函数）用于告知智能体：

* 其核心任务或目标
* 其个性或角色设定（例如"你是一个乐于助人的助手"、"你是个风趣的海盗"）
* 行为约束（例如"仅回答关于X的问题"、"绝不透露Y"）
* 如何使用其`tools`。应说明每个工具的用途及调用时机，补充工具自身的描述
* 期望的输出格式（例如"以JSON格式响应"、"提供带项目符号的列表"）

**有效指令的设计技巧：**

* **清晰具体：** 避免歧义，明确说明期望行为和结果
* **使用Markdown：** 对复杂指令使用标题、列表等提高可读性
* **提供示例（Few-Shot）：** 对于复杂任务或特定输出格式，直接在指令中包含示例
* **引导工具使用：** 不仅列出工具，还要解释*何时*及*为何*使用它们

```python
# Example: Adding instructions
capital_agent = LlmAgent(
    model="gemini-2.0-flash",
    name="capital_agent",
    description="Answers user questions about the capital city of a given country.",
    instruction="""You are an agent that provides the capital city of a country.
When a user asks for the capital of a country:
1. Identify the country name from the user's query.
2. Use the `get_capital_city` tool to find the capital.
3. Respond clearly to the user, stating the capital city.
Example Query: "What's the capital of France?"
Example Response: "The capital of France is Paris."
""",
    # tools will be added next
)
```

*（注：对于适用于系统中*所有*智能体的指令，可考虑在根智能体上使用`global_instruction`，详见[多智能体](multi-agents.md)章节。）*

## 装备智能体：工具（`tools`）

工具赋予`LlmAgent`超越大模型内置知识或推理的能力。它们使智能体能够与外部世界交互、执行计算、获取实时数据或完成特定操作。

* **`tools`（可选）：** 提供智能体可使用的工具列表。列表中的每个项目可以是：
    * Python函数（自动包装为`FunctionTool`）
    * 继承自`BaseTool`的类实例
    * 其他智能体实例（`AgentTool`，实现智能体间委派——参见[多智能体](multi-agents.md)）

大模型会基于对话内容和指令，根据函数/工具名称、描述（来自文档字符串或`description`字段）及参数模式来决定调用哪个工具。

```python
# Define a tool function
def get_capital_city(country: str) -> str:
  """Retrieves the capital city for a given country."""
  # Replace with actual logic (e.g., API call, database lookup)
  capitals = {"france": "Paris", "japan": "Tokyo", "canada": "Ottawa"}
  return capitals.get(country.lower(), f"Sorry, I don't know the capital of {country}.")

# Add the tool to the agent
capital_agent = LlmAgent(
    model="gemini-2.0-flash",
    name="capital_agent",
    description="Answers user questions about the capital city of a given country.",
    instruction="""You are an agent that provides the capital city of a country... (previous instruction text)""",
    tools=[get_capital_city] # Provide the function directly
)
```

更多工具相关信息请参阅[工具](../tools/index.md)章节。

## 高级配置与控制

除核心参数外，`LlmAgent`还提供多个选项进行精细控制：

### 微调大模型生成（`generate_content_config`）

可通过`generate_content_config`调整底层大模型的响应生成方式。

* **`generate_content_config`（可选）：** 传递`google.genai.types.GenerateContentConfig`实例以控制`temperature`（随机性）、`max_output_tokens`（响应长度）、`top_p`、`top_k`和安全设置等参数。

    ```python
    from google.genai import types

    agent = LlmAgent(
        # ... other params
        generate_content_config=types.GenerateContentConfig(
            temperature=0.2, # 更确定性的输出
            max_output_tokens=250
        )
    )
    ```

### 结构化数据（`input_schema`、`output_schema`、`output_key`）

对于需要结构化数据交换的场景，可使用Pydantic模型。

* **`input_schema`（可选）：** 定义表示预期输入结构的Pydantic `BaseModel`类。若设置，传递给该智能体的用户消息内容*必须*是符合此模式的JSON字符串。指令应相应引导用户或前置智能体。

* **`output_schema`（可选）：** 定义表示期望输出结构的Pydantic `BaseModel`类。若设置，智能体的最终响应*必须*是符合此模式的JSON字符串。
    * **限制：** 使用`output_schema`可在大模型内实现受控生成，但*会禁用智能体使用工具或转移控制权的能力*。指令必须引导大模型直接生成符合模式的JSON。

* **`output_key`（可选）：** 提供字符串键。若设置，智能体*最终*响应的文本内容将自动保存到会话状态字典中的该键下（例如`session.state[output_key] = agent_response_text`）。这在智能体间或工作流步骤间传递结果时非常有用。

```python
from pydantic import BaseModel, Field

class CapitalOutput(BaseModel):
    capital: str = Field(description="The capital of the country.")

structured_capital_agent = LlmAgent(
    # ... name, model, description
    instruction="""You are a Capital Information Agent. Given a country, respond ONLY with a JSON object containing the capital. Format: {"capital": "capital_name"}""",
    output_schema=CapitalOutput, # Enforce JSON output
    output_key="found_capital"  # Store result in state['found_capital']
    # Cannot use tools=[get_capital_city] effectively here
)
```

### 管理上下文（`include_contents`）

控制智能体是否接收先前的对话历史记录。

* **`include_contents`（可选，默认：`'default'`）：** 决定是否向大模型发送`contents`（历史记录）。
    * `'default'`：智能体接收相关对话历史记录
    * `'none'`：智能体不接收任何先前`contents`。仅基于当前指令和*当前*轮次提供的输入进行操作（适用于无状态任务或强制特定上下文场景）

    ```python
    stateless_agent = LlmAgent(
        # ... other params
        include_contents='none'
    )
    ```

### 规划与代码执行

对于涉及多步推理或执行代码的复杂场景：

* **`planner`（可选）：** 分配`BasePlanner`实例以实现执行前的多步推理和规划（参见[多智能体](multi-agents.md)模式）
* **`code_executor`（可选）：** 提供`BaseCodeExecutor`实例，允许智能体执行大模型响应中的代码块（如Python）（参见[工具/内置工具](../tools/built-in-tools.md)）

## 完整示例

??? "代码"
    以下是完整的`capital_agent`示例：

    ```python
    # 基础首都智能体的完整示例代码
    --8<-- "examples/python/snippets/agents/llm-agent/capital_agent.py"
    ```

*（此示例演示了核心概念。更复杂的智能体可能包含模式、上下文控制、规划等功能）*

## 相关概念（延展主题）

虽然本文涵盖`LlmAgent`的核心配置，但以下几个相关概念提供更高级的控制，详见其他文档：

* **回调函数：** 使用`before_model_callback`、`after_model_callback`等拦截执行点（模型调用前/后、工具调用前/后）。参见[回调函数](../callbacks/types-of-callbacks.md)
* **多智能体控制：** 智能体交互的高级策略，包括规划（`planner`）、控制智能体转移（`disallow_transfer_to_parent`、`disallow_transfer_to_peers`）和系统级指令（`global_instruction`）。参见[多智能体](multi-agents.md)