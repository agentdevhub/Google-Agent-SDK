# 函数工具

## 什么是函数工具？

当开箱即用的工具无法完全满足特定需求时，开发者可以创建自定义函数工具。这允许实现**定制化功能**，例如连接专有数据库或实现独特算法。

*例如*，一个名为"myfinancetool"的函数工具可能是计算特定财务指标的函数。ADK还支持长时间运行的函数，因此如果计算耗时较长，智能体可以继续处理其他任务。

ADK提供多种创建函数工具的方式，分别适用于不同复杂度和控制需求：

1. 函数工具
2. 长时间运行函数工具
3. 智能体即工具

## 1. 函数工具

将函数转化为工具是将自定义逻辑集成到智能体中的直接方式。这种方法提供了灵活性和快速集成能力。

### 参数

使用标准的**JSON可序列化类型**（如字符串、整数、列表、字典）定义函数参数。注意避免为参数设置默认值，因为大模型目前不支持解析默认值。

### 返回类型

Python函数工具的首选返回类型是**字典**。这允许通过键值对结构化响应，为大模型提供上下文和清晰度。如果函数返回非字典类型，框架会自动将其包装为包含单个键**"result"**的字典。

尽量使返回值具有描述性。*例如*，与其返回数字错误代码，不如返回包含"error_message"键的字典，其中包含人类可读的解释。**请记住**需要理解结果的是大模型而非代码。最佳实践是在返回字典中包含"status"键来指示整体结果（如"success"、"error"、"pending"），为大模型提供操作状态的明确信号。

### 文档字符串

函数的文档字符串作为工具描述发送给大模型。因此，编写完善全面的文档字符串对于大模型理解如何有效使用工具至关重要。需清晰说明函数用途、参数含义和预期返回值。

??? "示例"

    该工具是一个获取给定股票代码/符号股价的Python函数。

    <u>注意</u>：使用此工具前需要`pip install yfinance`库。

    ```py
    --8<-- "examples/python/snippets/tools/function-tools/func_tool.py"
    ```

    该工具的返回值将被包装成字典。

    ```json
    {"result": "$123"}
    ```

### 最佳实践

虽然定义函数时有很大灵活性，但请记住简单性可提升大模型的可用性。考虑以下准则：

* **参数越少越好**：减少参数数量以降低复杂度  
* **简单数据类型**：尽可能使用`str`和`int`等基本类型而非自定义类  
* **有意义的命名**：函数名和参数名显著影响大模型对工具的理解和使用。选择能清晰反映函数用途和输入含义的名称。避免使用`do_stuff()`等通用名称  

## 2. 长时间运行函数工具

专为需要大量处理时间但不会阻塞智能体执行的任务设计。该工具是`FunctionTool`的子类。

使用`LongRunningFunctionTool`时，Python函数可以启动长时间运行操作，并可选择返回**中间结果**以保持模型和用户了解进度。智能体可以继续处理其他任务。典型场景是需要人工审批才能继续执行的人机交互流程。

### 工作原理

使用`LongRunningFunctionTool`包装Python*生成器*函数（使用`yield`的函数）。

1. **初始化**：当大模型调用工具时，生成器函数开始执行

2. **中间更新(`yield`)**：函数应定期生成中间Python对象（通常是字典）报告进度。ADK框架获取每个生成值并将其包装在`FunctionResponse`中发回大模型，使大模型能通知用户（如状态、完成百分比、消息）

3. **完成(`return`)**：任务结束时，生成器函数使用`return`提供最终Python对象结果

4. **框架处理**：ADK框架管理执行过程。将每个生成值作为中间`FunctionResponse`发回。当生成器完成时，框架将返回值作为最终`FunctionResponse`的内容发送，向大模型标记长时间运行操作结束

### 创建工具

定义生成器函数并用`LongRunningFunctionTool`类包装：

```py
from google.adk.tools import LongRunningFunctionTool

# Define your generator function (see example below)
def my_long_task_generator(*args, **kwargs):
    # ... setup ...
    yield {"status": "pending", "message": "Starting task..."} # Framework sends this as FunctionResponse
    # ... perform work incrementally ...
    yield {"status": "pending", "progress": 50}               # Framework sends this as FunctionResponse
    # ... finish work ...
    return {"status": "completed", "result": "Final outcome"} # Framework sends this as final FunctionResponse

# Wrap the function
my_tool = LongRunningFunctionTool(func=my_long_task_generator)
```

### 中间更新

生成结构化Python对象（如字典）对提供有意义的更新至关重要。应包含以下键：

* status：如"pending"、"running"、"waiting_for_input"

* progress：如百分比、已完成步骤数

* message：面向用户/大模型的描述性文本

* estimated_completion_time：如可计算

框架将每个生成值打包成FunctionResponse发送给大模型。

### 最终结果

生成器函数返回的Python对象被视为工具执行的最终结果。框架将该值（即使为None）打包到发送给大模型的最终`FunctionResponse`内容中，标记工具执行完成。

??? "示例：文件处理模拟"

    ```py
    --8<-- "examples/python/snippets/tools/function-tools/file_processor.py"
    ```

#### 示例关键点

* **process_large_file**：该生成器模拟耗时操作，生成中间状态/进度字典

* **`LongRunningFunctionTool`**：包装生成器；框架处理发送生成的更新和最终返回值作为连续的FunctionResponse

* **智能体指令**：指导大模型使用工具并理解传入的FunctionResponse流（进度与完成）以更新用户

* **最终返回**：函数返回最终结果字典，该字典在结束FunctionResponse中发送以标记完成

## 3. 智能体即工具

这一强大功能允许通过将其他智能体作为工具调用来利用系统中的智能体能力。智能体即工具使您可以调用其他智能体执行特定任务，实现**责任委托**。概念上类似于创建调用其他智能体并将智能体响应作为函数返回值的Python函数。

### 与子智能体的关键区别

需区分智能体即工具与子智能体：

* **智能体即工具**：当智能体A将智能体B作为工具调用（使用智能体即工具）时，智能体B的答案**传回**智能体A，智能体A汇总答案并生成对用户的响应。智能体A保持控制权并继续处理后续用户输入  

* **子智能体**：当智能体A将智能体B作为子智能体调用时，应答用户的职责完全**转移给智能体B**。智能体A实际上退出交互循环。所有后续用户输入将由智能体B应答

### 使用方法

要将智能体作为工具使用，需用AgentTool类包装智能体。

```py
tools=[AgentTool(agent=agent_b)]
```

### 自定义

`AgentTool`类提供以下属性用于自定义行为：

* **skip_summarization: bool**：如设为True，框架将**跳过基于大模型的工具智能体响应摘要**。当工具响应已格式良好且无需进一步处理时很有用

??? "示例"

    ```py
    --8<-- "examples/python/snippets/tools/function-tools/summarizer.py"
    ```

### 工作原理

1. 当`main_agent`收到长文本时，其指令告知其对长文本使用'summarize'工具  
2. 框架识别'summarize'为包装`summary_agent`的`AgentTool`  
3. 后台中，`main_agent`将以长文本作为输入调用`summary_agent`  
4. `summary_agent`将根据其指令处理文本并生成摘要  
5. **`summary_agent`的响应随后传回`main_agent`**  
6. `main_agent`可获取摘要并形成对用户的最终响应（如"以下是文本摘要：..."）