# 制品（Artifacts）

在ADK中，**制品（Artifacts）**代表了一种关键机制，用于管理与特定用户交互会话或跨多个会话持久化关联的命名版本化二进制数据。它们使您的智能体和工具能够处理超越简单文本字符串的数据，实现涉及文件、图像、音频和其他二进制格式的更丰富交互。

## 什么是制品？

* **定义：** 制品本质上是一段二进制数据（如文件内容），在特定作用域（会话或用户）内通过唯一的`filename`字符串标识。每次使用相同文件名保存制品时，都会创建新版本。  

* **表示方式：** 制品始终使用标准`google.genai.types.Part`对象表示。核心数据通常存储在`Part`的`inline_data`属性中，该属性包含：  
    * `data`：作为`bytes`的原始二进制内容  
    * `mime_type`：表示数据类型（如`'image/png'`、`'application/pdf'`）的字符串。这对后续正确解析数据至关重要  

    ```py
    # 示例：如何将制品表示为types.Part
    import google.genai.types as types

    # 假设'image_bytes'包含PNG图像的二进制数据
    image_bytes = b'\x89PNG\r\n\x1a\n...' # 实际图像字节的占位符

    image_artifact = types.Part(
        inline_data=types.Blob(
            mime_type="image/png",
            data=image_bytes
        )
    )

    # 也可使用便捷构造函数：
    # image_artifact_alt = types.Part.from_data(data=image_bytes, mime_type="image/png")

    print(f"制品MIME类型: {image_artifact.inline_data.mime_type}")
    print(f"制品数据(前10字节): {image_artifact.inline_data.data[:10]}...")
    ```

* **持久化与管理：** 制品不直接存储在智能体或会话状态中。其存储和检索由专用**制品服务**（`BaseArtifactService`的实现，定义于`google.adk.artifacts.base_artifact_service.py`）管理。ADK提供如`InMemoryArtifactService`（用于测试/临时存储，定义于`google.adk.artifacts.in_memory_artifact_service.py`）和`GcsArtifactService`（使用Google Cloud Storage持久化存储，定义于`google.adk.artifacts.gcs_artifact_service.py`）等实现。所选服务在保存数据时会自动处理版本控制。

## 为何使用制品？

虽然会话`state`适合存储小型配置或对话上下文（如字符串、数字、布尔值或小型字典/列表），但制品专为涉及二进制或大型数据的场景设计：

1. **处理非文本数据：** 轻松存储和检索图像、音频片段、视频片段、PDF、电子表格或与智能体功能相关的任何其他文件格式  
2. **持久化大型数据：** 会话状态通常不适合存储大量数据。制品提供了专用机制来持久化大型数据块，而不会使会话状态混乱  
3. **用户文件管理：** 提供用户上传文件（可保存为制品）以及检索或下载智能体生成文件（从制品加载）的能力  
4. **共享输出：** 使工具或智能体能够生成二进制输出（如PDF报告或生成图像），这些输出可通过`save_artifact`保存，并可在应用程序其他部分甚至后续会话中访问（如果使用用户命名空间）  
5. **缓存二进制数据：** 将产生二进制数据的计算密集型操作结果（如渲染复杂图表图像）存储为制品，避免在后续请求中重新生成  

本质上，当您的智能体需要处理需要持久化、版本控制或共享的类文件二进制数据时，由`ArtifactService`管理的制品是ADK中的适当机制。

## 常见用例

制品为ADK应用程序中的二进制数据处理提供了灵活方式。以下是它们发挥价值的典型场景：

* **生成报告/文件：**
    * 工具或智能体生成报告（如PDF分析、CSV数据导出、图像图表）  
    * 工具使用`tool_context.save_artifact("monthly_report_oct_2024.pdf", report_part)`存储生成的文件  
    * 用户随后可要求智能体检索此报告，可能涉及另一个工具使用`tool_context.load_artifact("monthly_report_oct_2024.pdf")`或使用`tool_context.list_artifacts()`列出可用报告  

* **处理用户上传：**  
    * 用户通过前端界面上传文件（如用于分析的图像、用于摘要的文档）  
    * 应用程序后端接收文件，从其字节和MIME类型创建`types.Part`，并使用`runner.session_service`（或直接代理运行外的类似机制）或通过`context.save_artifact`在运行中的专用工具/回调来存储它，如果应跨会话持久化（如`user:uploaded_image.jpg`），则可能使用`user:`命名空间  
    * 然后可以提示智能体处理此上传文件，使用`context.load_artifact("user:uploaded_image.jpg")`检索它  

* **存储中间二进制结果：**  
    * 智能体执行复杂的多步骤流程，其中一步生成中间二进制数据（如音频合成、模拟结果）  
    * 使用`context.save_artifact`以临时或描述性名称（如`"temp_audio_step1.wav"`）保存此数据  
    * 流程中的后续智能体或工具（可能在`SequentialAgent`中或稍后触发）可以使用`context.load_artifact`加载此中间制品以继续流程  

* **持久化用户数据：**  
    * 存储非简单键值状态的用户特定配置或数据  
    * 智能体使用`context.save_artifact("user:profile_settings.json", settings_part)`或`context.save_artifact("user:avatar.png", avatar_part)`保存用户偏好或头像  
    * 这些制品可在该用户的任何未来会话中加载以个性化其体验  

* **缓存生成内容：**  
    * 智能体基于特定输入频繁生成相同二进制输出（如公司徽标图像、标准音频问候）  
    * 在生成前，`before_tool_callback`或`before_agent_callback`使用`context.load_artifact`检查制品是否存在  
    * 如果存在，则使用缓存制品，跳过生成步骤  
    * 如果不存在，则生成内容，并在`after_tool_callback`或`after_agent_callback`中调用`context.save_artifact`以缓存供下次使用  

## 核心概念

理解制品需要掌握几个关键组件：管理它们的服务、用于保存它们的数据结构以及它们的标识和版本控制方式。

### 制品服务（`BaseArtifactService`）

* **角色：** 负责制品实际存储和检索逻辑的核心组件。定义了制品*如何*和*何处*持久化  

* **接口：** 由抽象基类`BaseArtifactService`（`google.adk.artifacts.base_artifact_service.py`）定义。任何具体实现必须提供以下方法：  

    * `save_artifact(...) -> int`：存储制品数据并返回分配的版本号  
    * `load_artifact(...) -> Optional[types.Part]`：检索特定版本（或最新）制品  
    * `list_artifact_keys(...) -> list[str]`：列出给定作用域内制品的唯一文件名  
    * `delete_artifact(...) -> None`：删除制品（根据实现可能删除所有版本）  
    * `list_versions(...) -> list[int]`：列出特定制品文件名的所有可用版本号  

* **配置：** 初始化`Runner`时提供制品服务实例（如`InMemoryArtifactService`、`GcsArtifactService`）。`Runner`然后通过`InvocationContext`使此服务对智能体和工具可用  

```py
from google.adk.runners import Runner
from google.adk.artifacts import InMemoryArtifactService # Or GcsArtifactService
from google.adk.agents import LlmAgent # Any agent
from google.adk.sessions import InMemorySessionService

# Example: Configuring the Runner with an Artifact Service
my_agent = LlmAgent(name="artifact_user_agent", model="gemini-2.0-flash")
artifact_service = InMemoryArtifactService() # Choose an implementation
session_service = InMemorySessionService()

runner = Runner(
    agent=my_agent,
    app_name="my_artifact_app",
    session_service=session_service,
    artifact_service=artifact_service # Provide the service instance here
)
# Now, contexts within runs managed by this runner can use artifact methods
```

### 制品数据（`google.genai.types.Part`）

* **标准表示：** 制品内容统一使用`google.genai.types.Part`对象表示，该结构也用于LLM消息的部分  

* **关键属性（`inline_data`）：** 对于制品，最相关的属性是`inline_data`，它是一个`google.genai.types.Blob`对象，包含：  

    * `data`（`bytes`）：制品的原始二进制内容  
    * `mime_type`（`str`）：描述二进制数据性质的标准MIME类型字符串（如`'application/pdf'`、`'image/png'`、`'audio/mpeg'`）。**这在加载制品时对正确解析至关重要**  

* **创建：** 通常使用`from_data`类方法或直接构造`Blob`为制品创建`Part`  

```py
import google.genai.types as types

# Example: Creating an artifact Part from raw bytes
pdf_bytes = b'%PDF-1.4...' # Your raw PDF data
pdf_mime_type = "application/pdf"

# Using the constructor
pdf_artifact = types.Part(
    inline_data=types.Blob(data=pdf_bytes, mime_type=pdf_mime_type)
)

# Using the convenience class method (equivalent)
pdf_artifact_alt = types.Part.from_data(data=pdf_bytes, mime_type=pdf_mime_type)

print(f"Created artifact with MIME type: {pdf_artifact.inline_data.mime_type}")
```

### 文件名（`str`）

* **标识符：** 用于在其特定命名空间（见下文）内命名和检索制品的简单字符串  
* **唯一性：** 文件名在其作用域（会话或用户命名空间）内必须唯一  
* **最佳实践：** 使用描述性名称，可能包含文件扩展名（如`"monthly_report.pdf"`、`"user_avatar.jpg"`），尽管扩展名本身不决定行为——`mime_type`才决定  

### 版本控制（`int`）

* **自动版本控制：** 制品服务自动处理版本控制。调用`save_artifact`时，服务确定该文件名和作用域的下一个可用版本号（通常从0开始递增）  
* **由`save_artifact`返回：** `save_artifact`方法返回分配给新保存制品的整数版本号  
* **检索：**  
  * `load_artifact(..., version=None)`（默认）：检索制品的*最新*可用版本  
  * `load_artifact(..., version=N)`：检索特定版本`N`  
* **列出版本：** `list_versions`方法（在服务上，非上下文）可用于查找制品的所有现有版本号  

### 命名空间（会话与用户）

* **概念：** 制品可以限定于特定会话或更广泛地限定于应用程序中用户的所有会话。此作用域由`filename`格式确定，并由`ArtifactService`内部处理  

* **默认（会话作用域）：** 如果使用普通文件名（如`"report.pdf"`），制品与特定`app_name`、`user_id`和`session_id`关联。仅可在该确切会话上下文中访问  

  * 内部路径（示例）：`app_name/user_id/session_id/report.pdf/<version>`（如`GcsArtifactService._get_blob_name`和`InMemoryArtifactService._artifact_path`中所见）  

* **用户作用域（`"user:"`前缀）：** 如果文件名前缀为`"user:"`，如`"user:profile.png"`，制品仅与`app_name`和`user_id`关联。可以从属于该用户的应用程序中的*任何*会话访问或更新  

  * 内部路径（示例）：`app_name/user_id/user/user:profile.png/<version>`（`user:`前缀通常保留在最终路径段中以清晰，如服务实现中所见）  
  * **用例：** 理想用于属于用户本身、独立于特定对话的数据，如头像、用户偏好文件或长期报告  

```py
# Example illustrating namespace difference (conceptual)

# Session-specific artifact filename
session_report_filename = "summary.txt"

# User-specific artifact filename
user_config_filename = "user:settings.json"

# When saving 'summary.txt', it's tied to the current session ID.
# When saving 'user:settings.json', it's tied only to the user ID.
```

这些核心概念共同为ADK框架中的二进制数据管理提供了灵活系统。

## 与制品交互（通过上下文对象）

在智能体逻辑（特别是回调或工具中）与制品交互的主要方式是通过`CallbackContext`和`ToolContext`对象提供的方法。这些方法抽象了由`ArtifactService`管理的底层存储细节。

### 前提条件：配置`ArtifactService`

在使用上下文对象的任何制品方法之前，**必须**在初始化`Runner`时提供[`BaseArtifactService`实现](#available-implementations)（如[`InMemoryArtifactService`](#inmemoryartifactservice)或[`GcsArtifactService`](#gcsartifactservice)）的实例。

```py
from google.adk.runners import Runner
from google.adk.artifacts import InMemoryArtifactService # Or GcsArtifactService
from google.adk.agents import LlmAgent
from google.adk.sessions import InMemorySessionService

# Your agent definition
agent = LlmAgent(name="my_agent", model="gemini-2.0-flash")

# Instantiate the desired artifact service
artifact_service = InMemoryArtifactService()

# Provide it to the Runner
runner = Runner(
    agent=agent,
    app_name="artifact_app",
    session_service=InMemorySessionService(),
    artifact_service=artifact_service # Service must be provided here
)
```

如果`InvocationContext`中未配置`artifact_service`（未传递给`Runner`时发生），在上下文对象上调用`save_artifact`、`load_artifact`或`list_artifacts`将引发`ValueError`。

### 访问方法

制品交互方法可直接在`CallbackContext`（传递给智能体和模型回调）和`ToolContext`（传递给工具回调）实例上使用。记住`ToolContext`继承自`CallbackContext`。

#### 保存制品

* **方法：**

```py
context.save_artifact(filename: str, artifact: types.Part) -> int
```

* **可用上下文：** `CallbackContext`、`ToolContext`  

* **操作：**  

    1. 接受`filename`字符串（可包含`"user:"`前缀以用户作用域）和包含制品数据的`types.Part`对象（通常在`artifact.inline_data`中）  
    2. 将此信息传递给底层`artifact_service.save_artifact`  
    3. 服务存储数据，为该文件名和作用域分配下一个可用版本号  
    4. 关键的是，上下文通过向当前事件的`actions.artifact_delta`字典（定义于`google.adk.events.event_actions.py`）添加条目自动记录此操作。此增量将`filename`映射到新分配的`version`  

* **返回：** 分配给保存制品的整数`version`版本号  

* **代码示例（假设工具或回调中）：**

```py
import google.genai.types as types
from google.adk.agents.callback_context import CallbackContext # Or ToolContext

async def save_generated_report(context: CallbackContext, report_bytes: bytes):
    """Saves generated PDF report bytes as an artifact."""
    report_artifact = types.Part.from_data(
        data=report_bytes,
        mime_type="application/pdf"
    )
    filename = "generated_report.pdf"

    try:
        version = context.save_artifact(filename=filename, artifact=report_artifact)
        print(f"Successfully saved artifact '{filename}' as version {version}.")
        # The event generated after this callback will contain:
        # event.actions.artifact_delta == {"generated_report.pdf": version}
    except ValueError as e:
        print(f"Error saving artifact: {e}. Is ArtifactService configured?")
    except Exception as e:
        # Handle potential storage errors (e.g., GCS permissions)
        print(f"An unexpected error occurred during artifact save: {e}")

# --- Example Usage Concept ---
# report_data = b'...' # Assume this holds the PDF bytes
# await save_generated_report(callback_context, report_data)
```

#### 加载制品

* **方法：**

```py
context.load_artifact(filename: str, version: Optional[int] = None) -> Optional[types.Part]
```

* **可用上下文：** `CallbackContext`、`ToolContext`  

* **操作：**  

    1. 接受`filename`字符串（可包含`"user:"`）  
    2. 可选接受整数`version`。如果`version`为`None`（默认），则请求服务中的*最新*版本。如果提供特定整数，则请求该确切版本  
    3. 调用底层`artifact_service.load_artifact`  
    4. 服务尝试检索指定制品  

* **返回：** 如果找到，则包含制品数据的`types.Part`对象；如果制品（或指定版本）不存在，则为`None`  

* **代码示例（假设工具或回调中）：**

    ```py
    import google.genai.types as types
    from google.adk.agents.callback_context import CallbackContext # 或ToolContext

    async def process_latest_report(context: CallbackContext):
        """加载最新报告制品并处理其数据"""
        filename = "generated_report.pdf"
        try:
            # 加载最新版本
            report_artifact = context.load_artifact(filename=filename)

            if report_artifact and report_artifact.inline_data:
                print(f"成功加载最新制品'{filename}'")
                print(f"MIME类型: {report_artifact.inline_data.mime_type}")
                # 处理report_artifact.inline_data.data（字节）
                pdf_bytes = report_artifact.inline_data.data
                print(f"报告大小: {len(pdf_bytes)}字节")
                # ...进一步处理...
            else:
                print(f"未找到制品'{filename}'")

            # 示例：加载特定版本（如果版本0存在）
            # specific_version_artifact = context.load_artifact(filename=filename, version=0)
            # if specific_version_artifact:
            #     print(f"加载'{filename}'的版本0")

        except ValueError as e:
            print(f"加载制品错误: {e}。是否配置了ArtifactService？")
        except Exception as e:
            # 处理潜在存储错误
            print(f"制品加载期间发生意外错误: {e}")

    # --- 示例使用概念 ---
    # await process_latest_report(callback_context)
    ```

#### 列出制品文件名（仅工具上下文）

* **方法：**

```py
tool_context.list_artifacts() -> list[str]
```

* **可用上下文：** 仅`ToolContext`。此方法在基础`CallbackContext`上*不可用*  

* **操作：** 调用底层`artifact_service.list_artifact_keys`以获取当前作用域内所有唯一制品文件名（包括会话特定文件和前缀为`"user:"`的用户作用域文件）的列表  

* **返回：** 排序后的`list` `str`文件名  

* **代码示例（工具函数中）：**

```py
from google.adk.tools.tool_context import ToolContext

def list_user_files(tool_context: ToolContext) -> str:
    """Tool to list available artifacts for the user."""
    try:
        available_files = tool_context.list_artifacts()
        if not available_files:
            return "You have no saved artifacts."
        else:
            # Format the list for the user/LLM
            file_list_str = "\n".join([f"- {fname}" for fname in available_files])
            return f"Here are your available artifacts:\n{file_list_str}"
    except ValueError as e:
        print(f"Error listing artifacts: {e}. Is ArtifactService configured?")
        return "Error: Could not list artifacts."
    except Exception as e:
        print(f"An unexpected error occurred during artifact list: {e}")
        return "Error: An unexpected error occurred while listing artifacts."

# This function would typically be wrapped in a FunctionTool
# from google.adk.tools import FunctionTool
# list_files_tool = FunctionTool(func=list_user_files)
```

这些上下文方法提供了在ADK中管理二进制数据持久化的便捷一致方式，无论选择何种后端存储实现（`InMemoryArtifactService`、`GcsArtifactService`等）。

## 可用实现

ADK提供了`BaseArtifactService`接口的具体实现，提供适合不同开发阶段和部署需求的存储后端。这些实现根据`app_name`、`user_id`、`session_id`和`filename`（包括`user:`命名空间前缀）处理制品数据的存储、版本控制和检索细节。

### InMemoryArtifactService

* **源文件：** `google.adk.artifacts.in_memory_artifact_service.py`  
* **存储机制：** 使用Python字典（`self.artifacts`）存储在应用程序内存中来保存制品。字典键表示制品路径（包含应用、用户、会话/用户作用域和文件名），值是`types.Part`列表，其中每个元素对应一个版本（索引0是版本0，索引1是版本1，依此类推）  
* **关键特性：**  
    * **简单性：** 除核心ADK库外，无需外部设置或依赖  
    * **速度：** 操作通常非常快，因为它们涉及内存字典查找和列表操作  
    * **临时性：** 当运行应用程序的Python进程终止时，所有存储的制品都会**丢失**。数据在应用程序重启之间不会持久化  
* **用例：**  
    * 本地开发和测试的理想选择，不需要持久化  
    * 适用于短期演示或制品数据在单次应用程序运行中纯临时的场景  
* **实例化：**

```py
from google.adk.artifacts import InMemoryArtifactService

# Simply instantiate the class
in_memory_service = InMemoryArtifactService()

# Then pass it to the Runner
# runner = Runner(..., artifact_service=in_memory_service)
```

### GcsArtifactService

* **源文件：** `google.adk.artifacts.gcs_artifact_service.py`  
* **存储机制：** 利用Google Cloud Storage（GCS）进行持久制品存储。每个制品版本作为单独对象存储在指定GCS存储桶中  
* **对象命名约定：** 使用分层路径结构构造GCS对象名称（blob名称），通常：  
    * 会话作用域：`{app_name}/{user_id}/{session_id}/{filename}/{version}`  
    * 用户作用域：`{app_name}/{user_id}/user/{filename}/{version}`（注意：服务处理文件名中的`user:`前缀以确定路径结构）  
* **关键特性：**  
    * **持久性：** 存储在GCS中的制品在应用程序重启和部署之间持久存在  
    * **可扩展性：** 利用Google Cloud Storage的可扩展性和耐久性  
    * **版本控制：** 明确将每个版本存储为不同的GCS对象  
    * **需要配置：** 需要使用目标GCS `bucket_name`进行配置  
    * **需要权限：** 应用程序环境需要适当的凭据和IAM权限来读写指定的GCS存储桶  
* **用例：**  
    * 需要持久制品存储的生产环境  
    * 不同应用程序实例或服务需要共享制品的场景（通过访问相同GCS存储桶）  
    * 需要长期存储和检索用户或会话数据的应用程序  
* **实例化：**

```py
from google.adk.artifacts import GcsArtifactService

# Specify the GCS bucket name
gcs_bucket_name = "your-gcs-bucket-for-adk-artifacts" # Replace with your bucket name

try:
    gcs_service = GcsArtifactService(bucket_name=gcs_bucket_name)
    print(f"GcsArtifactService initialized for bucket: {gcs_bucket_name}")
    # Ensure your environment has credentials to access this bucket.
    # e.g., via Application Default Credentials (ADC)

    # Then pass it to the Runner
    # runner = Runner(..., artifact_service=gcs_service)

except Exception as e:
    # Catch potential errors during GCS client initialization (e.g., auth issues)
    print(f"Error initializing GcsArtifactService: {e}")
    # Handle the error appropriately - maybe fall back to InMemory or raise

```

选择合适的`ArtifactService`实现取决于应用程序对数据持久性、可扩展性和操作环境的需求。

## 最佳实践

为有效且可维护地使用制品：

* **选择正确服务：** 对快速原型设计、测试和不需要持久化的场景使用`InMemoryArtifactService`。对需要数据持久化和可扩展性的生产环境使用`GcsArtifactService`（或为其他后端实现自己的`BaseArtifactService`）  
* **有意义文件名：** 使用清晰、描述性文件名。包含相关扩展名（`.pdf`、`.png`、`.wav`）有助于人类理解内容，尽管`mime_type`决定程序处理方式。为临时与持久制品名称建立约定  
* **指定正确MIME类型：** 为`save_artifact`创建`types.Part`时始终提供准确的`mime_type`。这对应用程序或工具后续`load_artifact`正确解析`bytes`数据至关重要。尽可能使用标准IANA MIME类型  
* **理解版本控制：** 记住`load_artifact()`没有特定`version`参数时检索*最新*版本。如果逻辑依赖制品的特定历史版本，加载时务必提供整数版本号  
* **谨慎使用命名空间（`user:`）：** 仅当数据真正属于用户且应在其所有会话中可访问时，才对文件名使用`"user:"`前缀。对于特定对话或会话的数据，使用不带前缀的常规文件名  
* **错误处理：**  
    * 在调用上下文方法（`save_artifact`、`load_artifact`、`list_artifacts`）前始终检查是否配置了`artifact_service`——如果服务`None`，它们将引发`ValueError`。用`try...except ValueError`包装调用  
    * 检查`load_artifact`的返回值，因为如果制品或版本不存在，它将为`None`。不要假设它总是返回`Part`  
    * 准备处理底层存储服务的异常，特别是`GcsArtifactService`（如权限问题的`google.api_core.exceptions.Forbidden`、存储桶不存在的`NotFound`、网络错误）  
* **大小考虑：** 制品适合典型文件大小，但对极大文件要注意潜在成本和性能影响，特别是云存储。`InMemoryArtifactService`如果存储许多大型制品会消耗大量内存。评估极大数据是否更适合通过直接GCS链接或其他专用存储解决方案处理，而非在内存中传递整个字节数组  
* **清理策略：** 对于`GcsArtifactService`等持久存储，制品会一直存在直到显式删除。如果制品代表临时数据或有有限生命周期，实施清理策略。这可能涉及：  
    * 在存储桶上使用GCS生命周期策略  
    * 构建利用`artifact_service.delete_artifact`方法的特定工具或管理功能（注意：出于安全考虑，删除*不*通过上下文对象公开）  
    * 仔细管理文件名以允许基于模式的删除（如果需要）