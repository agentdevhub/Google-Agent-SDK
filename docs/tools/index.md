# 工具

## 什么是工具？

在ADK框架中，工具代表为AI智能体提供的特定能力，使其能够超越核心文本生成和推理能力，执行动作并与外部世界交互。强大智能体与基础语言模型的区别，往往在于其对工具的有效运用。

从技术角度看，工具通常是模块化的代码组件——**例如Python函数**、类方法，甚至是另一个专用智能体——用于执行明确定义的独立任务。这些任务通常涉及与外部系统或数据的交互。

<img src="../assets/agent-tool-call.png" alt="智能体工具调用示意图">

### 核心特性

**面向动作执行：** 工具执行的具体操作包括：
* 查询数据库
* 发起API请求（如获取天气数据、预约系统）
* 网络搜索
* 执行代码片段
* 从文档中检索信息（RAG）
* 与其他软件或服务交互

**扩展智能体能力：** 使智能体能够获取实时信息、影响外部系统，并突破其训练数据固有的知识局限。

**执行预定义逻辑：** 关键在于，工具执行的是开发者定义的特定逻辑。它们不像智能体核心大模型（LLM）那样具备独立推理能力。大模型负责判断使用哪个工具、何时使用以及输入参数，而工具本身仅执行其指定功能。

## 智能体如何使用工具

智能体通常通过函数调用等机制动态使用工具，流程一般包含以下步骤：

1. **推理分析：** 智能体的大模型解析系统指令、对话历史和用户请求
2. **工具选择：** 基于分析结果，大模型根据可用工具及其文档字符串决定是否调用工具
3. **调用执行：** 大模型生成所选工具所需参数并触发执行
4. **结果观察：** 智能体接收工具返回的输出结果
5. **最终处理：** 智能体将工具输出整合到推理流程中，形成后续响应、决定下一步操作或判断目标是否达成

可以将工具视为智能体核心（大模型）为完成复杂任务而随时调用的专用工具包。

## ADK中的工具类型

ADK通过支持多种工具类型提供灵活性：

1. **[函数工具](../tools/function-tools.md)：** 根据应用需求自定义的工具
    * **[函数/方法](../tools/function-tools.md#1-function-tool)：** 在代码中定义标准同步函数或方法（如Python def）
    * **[智能体即工具](../tools/function-tools.md#3-agent-as-a-tool)：** 将其他专用智能体作为父智能体的工具使用
    * **[长时运行函数工具](../tools/function-tools.md#2-long-running-function-tool)：** 支持异步操作或耗时较长的工具
2. **[内置工具](../tools/built-in-tools.md)：** 框架提供的开箱即用通用工具
        例如：谷歌搜索、代码执行、检索增强生成（RAG）
3. **[第三方工具](../tools/third-party-tools.md)：** 无缝集成主流外部库工具
        例如：LangChain工具、CrewAI工具

请访问上述链接文档页面获取各类工具的详细说明和示例。

## 在智能体指令中引用工具

在智能体指令中，可通过**函数名**直接引用工具。若工具的**函数名**和**文档字符串**描述充分，指令可重点说明**大模型应在何时使用该工具**，这能提升清晰度并帮助模型理解工具用途。

**必须明确指导智能体处理工具可能返回的不同值**。例如当工具返回错误信息时，应指定智能体应重试操作、放弃任务还是向用户请求更多信息。

此外，ADK支持工具串联使用，即一个工具的输出可作为另一个工具的输入。实现此类工作流时，需在智能体指令中**描述预期的工具使用顺序**以引导模型执行必要步骤。

### 示例

以下示例展示智能体如何通过**在指令中引用函数名**使用工具，同时演示如何引导智能体**处理工具返回的不同值**（如成功或错误信息），以及如何**协调多个工具的顺序使用**完成任务。

```py
--8<-- "examples/python/snippets/tools/overview/weather_sentiment.py"
```

## 工具上下文

对于高级场景，ADK允许通过在工具函数中添加特殊参数`tool_context: ToolContext`来访问额外上下文信息。当智能体执行过程中调用工具时，ADK将**自动**提供**ToolContext类实例**。

**ToolContext**提供以下关键信息和控制项：

* `state: State`：读取和修改当前会话状态，变更会被跟踪和持久化保存

* `actions: EventActions`：影响工具运行后智能体的后续动作（如跳过总结、转接其他智能体）

* `function_call_id: str`：框架为此特定工具调用分配的唯一标识符，可用于跟踪和认证响应关联。当单个模型响应中调用多个工具时尤为有用

* `function_call_event_id: str`：提供触发当前工具调用的**事件**唯一标识符，便于跟踪和日志记录

* `auth_response: Any`：若在此工具调用前完成认证流程，则包含认证响应/凭证

* 服务访问：与已配置服务（如Artifacts和Memory）交互的方法

### **状态管理**

`tool_context.state`属性提供对当前会话关联状态的直接读写访问。其行为类似字典，但能确保所有修改作为增量被跟踪并由会话服务持久化，使得工具能在不同交互和智能体步骤间维护共享信息。

* **读取状态**：使用标准字典访问方式（`tool_context.state['my_key']`）或`.get()`方法（`tool_context.state.get('my_key', default_value)`）

* **写入状态**：直接赋值（`tool_context.state['new_key'] = 'new_value'`），变更会记录在结果事件的state_delta中

* **状态前缀**：记住标准状态前缀：
    * `app:*`：应用所有用户共享
    * `user:*`：当前用户所有会话专用
    * （无前缀）：当前会话专用
    * `temp:*`：临时性，不跨调用持久化（适用于在单次运行调用内传递数据，但在工具上下文中通常用途有限）

```py
--8<-- "examples/python/snippets/tools/overview/user_preference.py"
```

### **控制智能体流程**

`tool_context.actions`属性持有**EventActions**对象。修改其属性可影响工具执行完成后智能体或框架的行为。

* **`skip_summarization: bool`**：（默认False）设为True时指示ADK跳过通常用于总结工具输出的LLM调用。当工具返回值已是用户就绪消息时特别有用

* **`transfer_to_agent: str`**：设置为其他智能体名称时，框架将停止当前智能体执行并**将会话控制权转移给指定智能体**，使工具能动态将任务交接给更专业的智能体

* **`escalate: bool`**：（默认False）设为True表示当前智能体无法处理请求，应向上级父智能体移交控制权（在层级结构中）。在LoopAgent中，子智能体工具设置**escalate=True**将终止循环

#### 示例

```py
--8<-- "examples/python/snippets/tools/overview/customer_support_agent.py"
```

##### 说明

* 定义两个智能体：`main_agent`和`support_agent`，其中`main_agent`设计为初始接触点
* 当`main_agent`调用`check_and_transfer`工具时，会检查用户查询
* 若查询含"urgent"字样，工具访问`tool_context`（特别是**`tool_context.actions`**）并将transfer_to_agent属性设为`support_agent`
* 该操作指示框架**将会话控制权转移给名为`support_agent`的智能体**
* 当`main_agent`处理紧急查询时，`check_and_transfer`工具触发转移，后续响应理想情况下应来自`support_agent`
* 对于不含紧急字样的普通查询，工具仅处理而不触发转移

此例展示工具如何通过ToolContext中的EventActions动态影响会话流向，将控制权转移给其他专业智能体。

### **认证机制**

ToolContext为需要与认证API交互的工具提供支持：

* **`auth_response`**：若框架在调用工具前已完成认证处理，则包含凭证（如令牌）（常见于RestApiTool和OpenAPI安全方案）

* **`request_credential(auth_config: dict)`**：当工具判定需要认证但凭证不可用时调用此方法，指示框架基于提供的auth_config启动认证流程

* **`get_auth_response()`**：在后续调用中（request_credential成功处理后）调用以获取用户提供的凭证

有关认证流程、配置和示例的详细说明，请参阅专门的工具认证文档页面。

### **上下文感知数据访问方法**

这些方法为工具提供与会话或用户关联持久化数据交互的便捷途径，由配置服务管理：

* **`list_artifacts()`**：返回artifact_service中当前为会话存储的所有文件（或键）名列表。工件通常是用户上传或由工具/智能体生成的文件（如图像、文档等）

* **`load_artifact(filename: str)`**：从**artifact_service**按文件名检索特定工件，可指定版本（默认返回最新版）。返回包含工件数据和MIME类型的`google.genai.types.Part`对象，未找到时返回None

* **`save_artifact(filename: str, artifact: types.Part)`**：向artifact_service保存工件新版本，返回新版本号（从0开始）

* **`search_memory(query: str)`**：使用配置的`memory_service`查询用户长期记忆，适用于从过往交互或存储知识中检索相关信息。**SearchMemoryResponse**结构取决于具体记忆服务实现，通常包含相关文本片段或对话摘录

#### 示例

```py
--8<-- "examples/python/snippets/tools/overview/doc_analysis.py"
```

通过利用**ToolContext**，开发者可创建更复杂、具备上下文感知的自定义工具，无缝集成ADK架构并增强智能体整体能力。

## 定义高效的工具函数

当使用标准Python函数作为ADK工具时，其定义方式显著影响智能体的正确使用能力。智能体的大模型（LLM）高度依赖函数的**名称**、**参数**、**类型提示**和**文档字符串**来理解其用途并生成正确调用。

以下是定义高效工具函数的关键准则：

* **函数命名：**
    * 使用能明确指示动作的描述性动名词组合（如`get_weather`、`search_documents`、`schedule_meeting`）
    * 避免通用名称（如`run`、`process`、`handle_data`）或过度模糊的名称（如`do_stuff`）。即使描述充分，像`do_stuff`这样的名称也可能让模型困惑何时使用该工具而非`cancel_flight`
    * 大模型在工具选择阶段将函数名作为主要标识符

* **参数设计：**
    * 函数可包含任意数量参数
    * 使用清晰描述性名称（如用`city`而非`c`，用`search_query`而非`q`）
    * **为所有参数提供类型提示**（如`city: str`、`user_id: int`、`items: list[str]`），这对ADK生成LLM正确模式至关重要
    * 确保所有参数类型**可JSON序列化**。标准Python类型（`str`、`int`、`float`、`bool`、`list`、`dict`及其组合）通常安全。避免直接使用复杂自定义类实例除非有明确JSON表示
    * **不要设置参数默认值**（如`def my_func(param1: str = "default")`）。底层模型生成函数调用时不可靠支持或使用默认值，所有必要信息应由LLM从上下文推导或显式请求

* **返回类型：**
    * 函数返回值**必须是字典（`dict`）**
    * 若函数返回非字典类型（如字符串、数字、列表），ADK框架会自动将其包装为`{'result': your_original_return_value}`格式再传回模型
    * 设计字典键值时应**便于LLM理解**，模型依赖此输出决定后续步骤
    * 包含有意义的键。例如不应仅返回`500`错误码，而应返回`{'status': 'error', 'error_message': 'Database connection failed'}`
    * **强烈建议**包含`status`键（如`'success'`、`'error'`、`'pending'`、`'ambiguous'`）以向模型清晰指示工具执行结果

* **文档字符串：**
    * **这是关键**。文档字符串是LLM获取描述信息的主要来源
    * **明确说明工具功能**，具体描述其用途和限制
    * **解释使用时机**，提供上下文或示例场景引导LLM决策
    * **清晰说明每个参数**，解释LLM需要为该参数提供哪些信息
    * 描述**`dict`返回值的结构和含义**，特别是不同的`status`值及相关数据键

    **良好定义示例：**

    ```python
    def lookup_order_status(order_id: str) -> dict:
      """根据订单ID查询客户订单当前状态。

      仅当用户明确询问特定订单状态且提供订单ID时使用本工具。
      不适用于一般查询。

      参数:
          order_id: 要查询订单的唯一标识符

      返回:
          包含订单状态的字典。
          可能状态：'已发货'、'处理中'、'待处理'、'错误'。
          成功示例：{'status': '已发货', 'tracking_number': '1Z9...'}
          错误示例：{'status': '错误', 'error_message': '未找到订单ID'}
      """
      # ... 从后端获取状态的实现代码 ...
      if status := fetch_status_from_backend(order_id):
           return {"status": status.state, "tracking_number": status.tracking} # 示例结构
      else:
           return {"status": "错误", "error_message": f"未找到订单ID {order_id}"}
    ```

* **简洁性与专注性：**
    * **保持工具专注**：每个工具理想情况下应执行一个明确定义的任务
    * **参数宜少不宜多**：模型通常能更可靠地处理参数较少且定义明确的工具
    * **使用简单数据类型**：尽可能选用基础类型（`str`、`int`、`bool`、`float`、`List[str]`等）而非复杂自定义类或深层嵌套结构作为参数
    * **分解复杂任务**：将执行多个逻辑步骤的函数拆分为更小更专注的工具。例如用`update_user_name(name: str)`、`update_user_address(address: str)`、`update_user_preferences(preferences: list[str])`等独立工具替代单一的`update_user_profile(profile: ProfileObject)`工具，便于LLM准确选择和使用功能

遵循这些准则可为LLM提供清晰结构和所需信息，使其能有效利用自定义函数工具，从而形成更强大可靠的智能体行为。