# 内置工具

这些内置工具提供开箱即用的功能（例如Google搜索或代码执行器），为智能体赋予通用能力。例如，需要从网络获取信息的智能体可以直接使用**google_search**工具，无需额外配置。

## 使用方法

1. **导入**：从`agents.tools`模块导入所需工具
2. **配置**：初始化工具，按需提供必要参数
3. **注册**：将初始化后的工具添加到智能体的**tools**列表中

工具注册后，智能体会根据**用户提示词**和其**指令**决定是否调用该工具。框架会在智能体调用时自动处理工具的执行流程。

## 可用内置工具

### Google搜索

`google_search`工具允许智能体通过Google Search执行网络搜索。该工具兼容Gemini 2模型，可直接添加到智能体工具列表。

```py
--8<-- "examples/python/snippets/tools/built-in-tools/google_search.py"
```

### 代码执行

`built_in_code_execution`工具支持智能体执行代码（需配合Gemini 2模型使用），可实现计算、数据处理或运行小型脚本等任务。

````py
--8<-- "examples/python/snippets/tools/built-in-tools/code_execution.py"
````

### Vertex AI搜索

`vertex_ai_search_tool`工具基于Google Cloud的Vertex AI Search技术，支持在私有配置的数据存储（如内部文档、公司政策、知识库）中进行检索。使用此内置工具需在配置时提供特定数据存储ID。

```py
--8<-- "examples/python/snippets/tools/built-in-tools/vertexai_search.py"
```

## 组合使用内置工具

以下代码示例演示如何组合多个内置工具，或通过多智能体模式将内置工具与其他工具配合使用：

```py
from google.adk.tools import agent_tool
from google.adk.agents import Agent
from google.adk.tools import google_search, built_in_code_execution

search_agent = Agent(
    model='gemini-2.0-flash',
    name='SearchAgent',
    instruction="""
    You're a specialist in Google Search
    """,
    tools=[google_search],
)
coding_agent = Agent(
    model='gemini-2.0-flash',
    name='CodeAgent',
    instruction="""
    You're a specialist in Code Execution
    """,
    tools=[built_in_code_execution],
)
root_agent = Agent(
    name="RootAgent",
    model="gemini-2.0-flash",
    description="Root Agent",
    tools=[agent_tool.AgentTool(agent=search_agent), agent_tool.AgentTool(agent=coding_agent)],
)
```

### 使用限制

!!! warning

    当前每个根智能体或独立智能体仅支持一个内置工具
    
例如以下在根智能体（或独立智能体）中使用两个及以上内置工具的方式**暂不支持**：

```py
root_agent = Agent(
    name="RootAgent",
    model="gemini-2.0-flash",
    description="Root Agent",
    tools=[built_in_code_execution, custom_function],
)
```

!!! warning

    内置工具无法在子智能体中使用
    
例如以下在子智能体中使用内置工具的方式**暂不支持**：

```py
search_agent = Agent(
    model='gemini-2.0-flash',
    name='SearchAgent',
    instruction="""
    You're a specialist in Google Search
    """,
    tools=[google_search],
)
coding_agent = Agent(
    model='gemini-2.0-flash',
    name='CodeAgent',
    instruction="""
    You're a specialist in Code Execution
    """,
    tools=[built_in_code_execution],
)
root_agent = Agent(
    name="RootAgent",
    model="gemini-2.0-flash",
    description="Root Agent",
    sub_agents=[
        search_agent,
        coding_agent
    ],
)
```