# ADK 中的多智能体系统

随着智能体应用复杂度的提升，将其构建为单一整体式智能体会导致开发、维护和逻辑梳理的困难。Agent Development Kit (ADK) 支持通过组合多个独立的 `BaseAgent` 实例来构建**多智能体系统 (MAS)**，从而开发复杂应用。

在 ADK 中，多智能体系统是由不同智能体（通常形成层级结构）通过协作实现更大目标的应用程序。这种架构方式具有显著优势，包括增强的模块化、专业化、可重用性、可维护性，以及通过专用工作流智能体定义结构化控制流的能力。

您可以通过组合派生自 `BaseAgent` 的多种智能体来构建这些系统：

* **大模型智能体**：基于大模型驱动的智能体（参见[大模型智能体](llm-agents.md)）
* **工作流智能体**：专门用于管理子智能体执行流程的特殊智能体（`SequentialAgent`、`ParallelAgent`、`LoopAgent`）（参见[工作流智能体](workflow-agents/index.md)）
* **自定义智能体**：继承自 `BaseAgent` 的自定义智能体，包含非大模型的专用逻辑（参见[自定义智能体](custom-agents.md)）

以下章节详细介绍了 ADK 的核心原语——如智能体层级结构、工作流智能体和交互机制——这些原语能帮助您有效构建和管理多智能体系统。

## 2. 用于智能体组合的 ADK 原语

ADK 提供核心构建块（原语），用于构建和管理多智能体系统中的交互。

### 2.1. 智能体层级结构（`parent_agent`、`sub_agents`）

多智能体系统的基础结构是通过 `BaseAgent` 中定义的父子关系建立的。

* **建立层级**：在初始化父智能体时，通过向 `sub_agents` 参数传递智能体实例列表来创建树形结构。ADK 在初始化期间（`google.adk.agents.base_agent.py` - `model_post_init`）会自动设置每个子智能体的 `parent_agent` 属性。
* **单一父节点规则**：一个智能体实例只能作为一个子智能体被添加一次。尝试分配第二个父节点将导致 `ValueError`。
* **重要性**：该层级结构定义了[工作流智能体](#22-作为协调器的工作流智能体)的作用范围，并影响大模型驱动委派的潜在目标。您可以使用 `agent.parent_agent` 导航层级结构，或使用 `agent.find_agent(name)` 查找后代。

```python
# Conceptual Example: Defining Hierarchy
from google.adk.agents import LlmAgent, BaseAgent

# Define individual agents
greeter = LlmAgent(name="Greeter", model="gemini-2.0-flash")
task_doer = BaseAgent(name="TaskExecutor") # Custom non-LLM agent

# Create parent agent and assign children via sub_agents
coordinator = LlmAgent(
    name="Coordinator",
    model="gemini-2.0-flash",
    description="I coordinate greetings and tasks.",
    sub_agents=[ # Assign sub_agents here
        greeter,
        task_doer
    ]
)

# Framework automatically sets:
# assert greeter.parent_agent == coordinator
# assert task_doer.parent_agent == coordinator

```

### 2.2. 作为协调器的工作流智能体

ADK 包含派生自 `BaseAgent` 的专用智能体，它们本身不执行任务，而是协调其 `sub_agents` 的执行流程。

* **[`SequentialAgent`](workflow-agents/sequential-agents.md)**：按列表顺序依次执行其 `sub_agents`。
    * **上下文**：依次传递*相同的* [`InvocationContext`](../runtime/index.md)，允许智能体通过共享状态轻松传递结果。

    ```python
    # 概念示例：顺序管道
    from google.adk.agents import SequentialAgent, LlmAgent

    step1 = LlmAgent(name="Step1_Fetch", output_key="data") # 将输出保存到 state['data']
    step2 = LlmAgent(name="Step2_Process", instruction="Process data from state key 'data'.")

    pipeline = SequentialAgent(name="MyPipeline", sub_agents=[step1, step2])
    # 当管道运行时，Step2 可以访问 Step1 设置的 state['data']。
    ```

* **[`ParallelAgent`](workflow-agents/parallel-agents.md)**：并行执行其 `sub_agents`。来自子智能体的事件可能会交错。
    * **上下文**：为每个子智能体修改 `InvocationContext.branch`（例如 `ParentBranch.ChildName`），提供不同的上下文路径，这有助于在某些内存实现中隔离历史记录。
    * **状态**：尽管分支不同，所有并行子智能体访问*相同的共享* `session.state`，使它们能够读取初始状态并写入结果（使用不同的键以避免竞争条件）。

    ```python
    # 概念示例：并行执行
    from google.adk.agents import ParallelAgent, LlmAgent

    fetch_weather = LlmAgent(name="WeatherFetcher", output_key="weather")
    fetch_news = LlmAgent(name="NewsFetcher", output_key="news")

    gatherer = ParallelAgent(name="InfoGatherer", sub_agents=[fetch_weather, fetch_news])
    # 当 gatherer 运行时，WeatherFetcher 和 NewsFetcher 并发运行。
    # 后续智能体可以读取 state['weather'] 和 state['news']。
    ```

* **[`LoopAgent`](workflow-agents/loop-agents.md)**：在循环中依次执行其 `sub_agents`。
    * **终止**：如果达到可选的 `max_iterations`，或任何子智能体生成带有 `actions.escalate=True` 的 [`Event`](../events/index.md)，则循环停止。
    * **上下文和状态**：在每次迭代中传递*相同的* `InvocationContext`，允许状态更改（例如计数器、标志）在循环之间持续存在。

    ```python
    # 概念示例：带条件的循环
    from google.adk.agents import LoopAgent, LlmAgent, BaseAgent
    from google.adk.events import Event, EventActions
    from google.adk.agents.invocation_context import InvocationContext
    from typing import AsyncGenerator

    class CheckCondition(BaseAgent): # 检查状态的自定义智能体
        async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
            status = ctx.session.state.get("status", "pending")
            is_done = (status == "completed")
            yield Event(author=self.name, actions=EventActions(escalate=is_done)) # 如果完成则升级

    process_step = LlmAgent(name="ProcessingStep") # 可能更新 state['status'] 的智能体

    poller = LoopAgent(
        name="StatusPoller",
        max_iterations=10,
        sub_agents=[process_step, CheckCondition(name="Checker")]
    )
    # 当 poller 运行时，它会重复执行 process_step 然后 Checker，
    # 直到 Checker 升级（state['status'] == 'completed'）或达到 10 次迭代。
    ```

### 2.3. 交互与通信机制

系统中的智能体通常需要交换数据或触发彼此的操作。ADK 通过以下方式实现这一点：

#### a) 共享会话状态（`session.state`）

在同一调用中运行的智能体（通过 `InvocationContext` 共享相同的 [`Session`](../sessions/session.md) 对象）进行被动通信的最基本方式。

* **机制**：一个智能体（或其工具/回调）写入值（`context.state['data_key'] = processed_data`），后续智能体读取该值（`data = context.state.get('data_key')`）。状态更改通过 [`CallbackContext`](../callbacks/index.md) 跟踪。
* **便利性**：[`LlmAgent`](llm-agents.md) 上的 `output_key` 属性会自动将智能体的最终响应文本（或结构化输出）保存到指定的状态键。
* **特性**：异步、被动通信。适用于由 `SequentialAgent` 协调的管道或在 `LoopAgent` 迭代之间传递数据。
* **另请参阅**：[状态管理](../sessions/state.md)

```python
# Conceptual Example: Using output_key and reading state
from google.adk.agents import LlmAgent, SequentialAgent

agent_A = LlmAgent(name="AgentA", instruction="Find the capital of France.", output_key="capital_city")
agent_B = LlmAgent(name="AgentB", instruction="Tell me about the city stored in state key 'capital_city'.")

pipeline = SequentialAgent(name="CityInfo", sub_agents=[agent_A, agent_B])
# AgentA runs, saves "Paris" to state['capital_city'].
# AgentB runs, its instruction processor reads state['capital_city'] to get "Paris".
```

#### b) 大模型驱动委派（智能体转移）

利用 [`LlmAgent`](llm-agents.md) 的理解能力，将任务动态路由到层级结构中的其他合适智能体。

* **机制**：智能体的大模型生成特定函数调用：`transfer_to_agent(agent_name='target_agent_name')`。
* **处理**：默认情况下，当存在子智能体或未禁止转移时，`AutoFlow` 会拦截此调用。它使用 `root_agent.find_agent()` 识别目标智能体，并更新 `InvocationContext` 以切换执行焦点。
* **要求**：调用的 `LlmAgent` 需要关于何时转移的清晰 `instructions`，潜在目标智能体需要不同的 `description` 以便大模型做出明智决策。转移范围（父节点、子智能体、同级节点）可在 `LlmAgent` 上配置。
* **特性**：基于大模型解释的动态、灵活路由。

```python
# Conceptual Setup: LLM Transfer
from google.adk.agents import LlmAgent

booking_agent = LlmAgent(name="Booker", description="Handles flight and hotel bookings.")
info_agent = LlmAgent(name="Info", description="Provides general information and answers questions.")

coordinator = LlmAgent(
    name="Coordinator",
    instruction="You are an assistant. Delegate booking tasks to Booker and info requests to Info.",
    description="Main coordinator.",
    # AutoFlow is typically used implicitly here
    sub_agents=[booking_agent, info_agent]
)
# If coordinator receives "Book a flight", its LLM should generate:
# FunctionCall(name='transfer_to_agent', args={'agent_name': 'Booker'})
# ADK framework then routes execution to booking_agent.
```

#### c) 显式调用（`AgentTool`）

允许 [`LlmAgent`](llm-agents.md) 将另一个 `BaseAgent` 实例视为可调用函数或[工具](../tools/index.md)。

* **机制**：将目标智能体实例包装在 `AgentTool` 中，并将其包含在父 `LlmAgent` 的 `tools` 列表中。`AgentTool` 会为大模型生成相应的函数声明。
* **处理**：当父大模型生成针对 `AgentTool` 的函数调用时，框架执行 `AgentTool.run_async`。此方法运行目标智能体，捕获其最终响应，将任何状态/工件更改转发回父上下文，并将响应作为工具结果返回。
* **特性**：同步（在父流程内）、显式、受控调用，类似于其他工具。
* **（注意**：需要显式导入和使用 `AgentTool`）。

```python
# Conceptual Setup: Agent as a Tool
from google.adk.agents import LlmAgent, BaseAgent
from google.adk.tools import agent_tool
from pydantic import BaseModel

# Define a target agent (could be LlmAgent or custom BaseAgent)
class ImageGeneratorAgent(BaseAgent): # Example custom agent
    name: str = "ImageGen"
    description: str = "Generates an image based on a prompt."
    # ... internal logic ...
    async def _run_async_impl(self, ctx): # Simplified run logic
        prompt = ctx.session.state.get("image_prompt", "default prompt")
        # ... generate image bytes ...
        image_bytes = b"..."
        yield Event(author=self.name, content=types.Content(parts=[types.Part.from_bytes(image_bytes, "image/png")]))

image_agent = ImageGeneratorAgent()
image_tool = agent_tool.AgentTool(agent=image_agent) # Wrap the agent

# Parent agent uses the AgentTool
artist_agent = LlmAgent(
    name="Artist",
    model="gemini-2.0-flash",
    instruction="Create a prompt and use the ImageGen tool to generate the image.",
    tools=[image_tool] # Include the AgentTool
)
# Artist LLM generates a prompt, then calls:
# FunctionCall(name='ImageGen', args={'image_prompt': 'a cat wearing a hat'})
# Framework calls image_tool.run_async(...), which runs ImageGeneratorAgent.
# The resulting image Part is returned to the Artist agent as the tool result.
```

这些原语提供了灵活性，可以设计从紧密耦合的顺序工作流到动态的大模型驱动委派网络的多智能体交互。

## 3. 使用 ADK 原语的常见多智能体模式

通过组合 ADK 的组合原语，您可以实现多种既定的多智能体协作模式。

### 协调器/调度器模式

* **结构**：一个中央 [`LlmAgent`](llm-agents.md)（协调器）管理多个专用 `sub_agents`。
* **目标**：将传入请求路由到适当的专业智能体。
* **使用的 ADK 原语**：
    * **层级结构**：协调器在 `sub_agents` 中列出专业智能体。
    * **交互**：主要使用**大模型驱动委派**（需要子智能体的清晰 `description` 和协调器上的适当 `instruction`）或**显式调用（`AgentTool`）**（协调器在其 `tools` 中包含 `AgentTool` 包装的专业智能体）。

```python
# Conceptual Code: Coordinator using LLM Transfer
from google.adk.agents import LlmAgent

billing_agent = LlmAgent(name="Billing", description="Handles billing inquiries.")
support_agent = LlmAgent(name="Support", description="Handles technical support requests.")

coordinator = LlmAgent(
    name="HelpDeskCoordinator",
    model="gemini-2.0-flash",
    instruction="Route user requests: Use Billing agent for payment issues, Support agent for technical problems.",
    description="Main help desk router.",
    # allow_transfer=True is often implicit with sub_agents in AutoFlow
    sub_agents=[billing_agent, support_agent]
)
# User asks "My payment failed" -> Coordinator's LLM should call transfer_to_agent(agent_name='Billing')
# User asks "I can't log in" -> Coordinator's LLM should call transfer_to_agent(agent_name='Support')
```

### 顺序管道模式

* **结构**：一个 [`SequentialAgent`](workflow-agents/sequential-agents.md) 包含按固定顺序执行的 `sub_agents`。
* **目标**：实现多步骤流程，其中一步的输出作为下一步的输入。
* **使用的 ADK 原语**：
    * **工作流**：`SequentialAgent` 定义顺序。
    * **通信**：主要使用**共享会话状态**。早期智能体写入结果（通常通过 `output_key`），后续智能体从 `context.state` 读取这些结果。

```python
# Conceptual Code: Sequential Data Pipeline
from google.adk.agents import SequentialAgent, LlmAgent

validator = LlmAgent(name="ValidateInput", instruction="Validate the input.", output_key="validation_status")
processor = LlmAgent(name="ProcessData", instruction="Process data if state key 'validation_status' is 'valid'.", output_key="result")
reporter = LlmAgent(name="ReportResult", instruction="Report the result from state key 'result'.")

data_pipeline = SequentialAgent(
    name="DataPipeline",
    sub_agents=[validator, processor, reporter]
)
# validator runs -> saves to state['validation_status']
# processor runs -> reads state['validation_status'], saves to state['result']
# reporter runs -> reads state['result']
```

### 并行扇出/收集模式

* **结构**：一个 [`ParallelAgent`](workflow-agents/parallel-agents.md) 并发运行多个 `sub_agents`，通常后跟一个稍后的智能体（在 `SequentialAgent` 中）聚合结果。
* **目标**：同时执行独立任务以减少延迟，然后合并它们的输出。
* **使用的 ADK 原语**：
    * **工作流**：`ParallelAgent` 用于并发执行（扇出）。通常嵌套在 `SequentialAgent` 中以处理后续聚合步骤（收集）。
    * **通信**：子智能体将结果写入**共享会话状态**中的不同键。后续的“收集”智能体读取多个状态键。

```python
# Conceptual Code: Parallel Information Gathering
from google.adk.agents import SequentialAgent, ParallelAgent, LlmAgent

fetch_api1 = LlmAgent(name="API1Fetcher", instruction="Fetch data from API 1.", output_key="api1_data")
fetch_api2 = LlmAgent(name="API2Fetcher", instruction="Fetch data from API 2.", output_key="api2_data")

gather_concurrently = ParallelAgent(
    name="ConcurrentFetch",
    sub_agents=[fetch_api1, fetch_api2]
)

synthesizer = LlmAgent(
    name="Synthesizer",
    instruction="Combine results from state keys 'api1_data' and 'api2_data'."
)

overall_workflow = SequentialAgent(
    name="FetchAndSynthesize",
    sub_agents=[gather_concurrently, synthesizer] # Run parallel fetch, then synthesize
)
# fetch_api1 and fetch_api2 run concurrently, saving to state.
# synthesizer runs afterwards, reading state['api1_data'] and state['api2_data'].
```

### 分层任务分解

* **结构**：多级智能体树，其中高级智能体分解复杂目标并将子任务委派给低级智能体。
* **目标**：通过递归分解复杂问题为更简单的可执行步骤来解决问题。
* **使用的 ADK 原语**：
    * **层级结构**：多级 `parent_agent`/`sub_agents` 结构。
    * **交互**：父智能体主要使用**大模型驱动委派**或**显式调用（`AgentTool`）**将任务分配给子智能体。结果通过工具响应或状态返回层级结构。

```python
# Conceptual Code: Hierarchical Research Task
from google.adk.agents import LlmAgent
from google.adk.tools import agent_tool

# Low-level tool-like agents
web_searcher = LlmAgent(name="WebSearch", description="Performs web searches for facts.")
summarizer = LlmAgent(name="Summarizer", description="Summarizes text.")

# Mid-level agent combining tools
research_assistant = LlmAgent(
    name="ResearchAssistant",
    model="gemini-2.0-flash",
    description="Finds and summarizes information on a topic.",
    tools=[agent_tool.AgentTool(agent=web_searcher), agent_tool.AgentTool(agent=summarizer)]
)

# High-level agent delegating research
report_writer = LlmAgent(
    name="ReportWriter",
    model="gemini-2.0-flash",
    instruction="Write a report on topic X. Use the ResearchAssistant to gather information.",
    tools=[agent_tool.AgentTool(agent=research_assistant)]
    # Alternatively, could use LLM Transfer if research_assistant is a sub_agent
)
# User interacts with ReportWriter.
# ReportWriter calls ResearchAssistant tool.
# ResearchAssistant calls WebSearch and Summarizer tools.
# Results flow back up.
```

### 审查/批评模式（生成器-批评器）

* **结构**：通常在 [`SequentialAgent`](workflow-agents/sequential-agents.md) 中包含两个智能体：生成器和批评器/审查器。
* **目标**：通过专用智能体审查生成的输出来提高其质量或有效性。
* **使用的 ADK 原语**：
    * **工作流**：`SequentialAgent` 确保生成先于审查。
    * **通信**：**共享会话状态**（生成器使用 `output_key` 保存输出；审查器读取该状态键）。审查器可能将其反馈保存到另一个状态键以供后续步骤使用。

```python
# Conceptual Code: Generator-Critic
from google.adk.agents import SequentialAgent, LlmAgent

generator = LlmAgent(
    name="DraftWriter",
    instruction="Write a short paragraph about subject X.",
    output_key="draft_text"
)

reviewer = LlmAgent(
    name="FactChecker",
    instruction="Review the text in state key 'draft_text' for factual accuracy. Output 'valid' or 'invalid' with reasons.",
    output_key="review_status"
)

# Optional: Further steps based on review_status

review_pipeline = SequentialAgent(
    name="WriteAndReview",
    sub_agents=[generator, reviewer]
)
# generator runs -> saves draft to state['draft_text']
# reviewer runs -> reads state['draft_text'], saves status to state['review_status']
```

### 迭代优化模式

* **结构**：使用 [`LoopAgent`](workflow-agents/loop-agents.md)，其中包含一个或多个智能体在多次迭代中处理任务。
* **目标**：逐步改进存储在会话状态中的结果（例如代码、文本、计划），直到达到质量阈值或最大迭代次数。
* **使用的 ADK 原语**：
    * **工作流**：`LoopAgent` 管理重复。
    * **通信**：**共享会话状态**对于智能体读取前一次迭代的输出并保存优化版本至关重要。
    * **终止**：循环通常基于 `max_iterations` 或专用检查智能体在结果满意时设置 `actions.escalate=True` 结束。

```python
# Conceptual Code: Iterative Code Refinement
from google.adk.agents import LoopAgent, LlmAgent, BaseAgent
from google.adk.events import Event, EventActions
from google.adk.agents.invocation_context import InvocationContext
from typing import AsyncGenerator

# Agent to generate/refine code based on state['current_code'] and state['requirements']
code_refiner = LlmAgent(
    name="CodeRefiner",
    instruction="Read state['current_code'] (if exists) and state['requirements']. Generate/refine Python code to meet requirements. Save to state['current_code'].",
    output_key="current_code" # Overwrites previous code in state
)

# Agent to check if the code meets quality standards
quality_checker = LlmAgent(
    name="QualityChecker",
    instruction="Evaluate the code in state['current_code'] against state['requirements']. Output 'pass' or 'fail'.",
    output_key="quality_status"
)

# Custom agent to check the status and escalate if 'pass'
class CheckStatusAndEscalate(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
        status = ctx.session.state.get("quality_status", "fail")
        should_stop = (status == "pass")
        yield Event(author=self.name, actions=EventActions(escalate=should_stop))

refinement_loop = LoopAgent(
    name="CodeRefinementLoop",
    max_iterations=5,
    sub_agents=[code_refiner, quality_checker, CheckStatusAndEscalate(name="StopChecker")]
)
# Loop runs: Refiner -> Checker -> StopChecker
# State['current_code'] is updated each iteration.
# Loop stops if QualityChecker outputs 'pass' (leading to StopChecker escalating) or after 5 iterations.
```

### 人在回路模式

* **结构**：在智能体工作流中集成人工干预点。
* **目标**：允许人工监督、批准、纠正或执行 AI 无法完成的任务。
* **使用的 ADK 原语（概念性）**：
    * **交互**：可以使用自定义**工具**实现，该工具暂停执行并向外部系统（例如 UI、工单系统）发送请求，等待人工输入。然后工具将人工响应返回给智能体。
    * **工作流**：可以使用**大模型驱动委派**（`transfer_to_agent`）针对触发外部工作流的概念性“人工智能体”，或在 `LlmAgent` 中使用自定义工具。
    * **状态/回调**：状态可以保存任务的详细信息供人工使用；回调可以管理交互流程。
    * **注意**：ADK 没有内置的“人工智能体”类型，因此需要自定义集成。

```python
# Conceptual Code: Using a Tool for Human Approval
from google.adk.agents import LlmAgent, SequentialAgent
from google.adk.tools import FunctionTool

# --- Assume external_approval_tool exists ---
# This tool would:
# 1. Take details (e.g., request_id, amount, reason).
# 2. Send these details to a human review system (e.g., via API).
# 3. Poll or wait for the human response (approved/rejected).
# 4. Return the human's decision.
# async def external_approval_tool(amount: float, reason: str) -> str: ...
approval_tool = FunctionTool(func=external_approval_tool)

# Agent that prepares the request
prepare_request = LlmAgent(
    name="PrepareApproval",
    instruction="Prepare the approval request details based on user input. Store amount and reason in state.",
    # ... likely sets state['approval_amount'] and state['approval_reason'] ...
)

# Agent that calls the human approval tool
request_approval = LlmAgent(
    name="RequestHumanApproval",
    instruction="Use the external_approval_tool with amount from state['approval_amount'] and reason from state['approval_reason'].",
    tools=[approval_tool],
    output_key="human_decision"
)

# Agent that proceeds based on human decision
process_decision = LlmAgent(
    name="ProcessDecision",
    instruction="Check state key 'human_decision'. If 'approved', proceed. If 'rejected', inform user."
)

approval_workflow = SequentialAgent(
    name="HumanApprovalWorkflow",
    sub_agents=[prepare_request, request_approval, process_decision]
)
```

这些模式为构建多智能体系统提供了起点。您可以根据需要混合搭配它们，为特定应用创建最有效的架构。