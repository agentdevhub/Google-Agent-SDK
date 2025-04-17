# OpenAPI 集成

## 通过 OpenAPI 集成 REST API

ADK 通过直接从 [OpenAPI 规范 (v3.x)](https://swagger.io/specification/) 自动生成可调用工具，简化了与外部 REST API 的交互过程。这消除了为每个 API 端点手动定义单独功能工具的需求。

!!! tip "核心优势"
    使用 `OpenAPIToolset` 可以立即从现有 API 文档（OpenAPI 规范）创建代理工具（`RestApiTool`），使代理能够无缝调用您的 Web 服务。

## 核心组件

* **`OpenAPIToolset`**：这是您将使用的主要类。通过 OpenAPI 规范初始化后，它会自动处理规范的解析和工具生成。
* **`RestApiTool`**：这个类代表单个可调用的 API 操作（如 `GET /pets/{petId}` 或 `POST /pets`）。`OpenAPIToolset` 会为规范中定义的每个操作创建一个 `RestApiTool` 实例。

## 工作原理

当使用 `OpenAPIToolset` 时，整个过程包含以下主要步骤：

1. **初始化与解析**：
    * 您可以通过 Python 字典、JSON 字符串或 YAML 字符串的形式向 `OpenAPIToolset` 提供 OpenAPI 规范。
    * 工具集内部会解析规范，解析所有内部引用（`$ref`）以理解完整的 API 结构。

2. **操作发现**：
    * 识别规范中定义的所有有效 API 操作（例如 `GET`、`POST`、`PUT`、`DELETE`）。

3. **工具生成**：
    * 对于每个发现的操作，`OpenAPIToolset` 会自动创建对应的 `RestApiTool` 实例。
    * **工具名称**：从规范中的 `operationId` 派生（转换为 `snake_case`，最多 60 个字符）。如果缺少 `operationId`，则根据方法和路径生成名称。
    * **工具描述**：使用操作的 `summary` 或 `description` 作为大模型的提示词。
    * **API 详情**：内部存储所需的 HTTP 方法、路径、服务器基础 URL、参数（路径、查询、头部、cookie）以及请求体模式。

4. **`RestApiTool` 功能**：每个生成的 `RestApiTool`：
    * **模式生成**：根据操作的参数和请求体动态创建 `FunctionDeclaration`。该模式会告知大模型如何调用工具（预期参数）。
    * **执行**：当大模型调用时，它会使用大模型提供的参数和 OpenAPI 规范中的详情构建正确的 HTTP 请求（URL、头部、查询参数、请求体）。它会处理身份验证（如果已配置）并使用 `requests` 库执行 API 调用。
    * **响应处理**：将 API 响应（通常是 JSON）返回给代理流程。

5. **身份验证**：您可以在初始化 `OpenAPIToolset` 时配置全局身份验证（如 API 密钥或 OAuth，详见[身份验证](../tools/authentication.md)）。此身份验证配置会自动应用于所有生成的 `RestApiTool` 实例。

## 使用流程

按照以下步骤将 OpenAPI 规范集成到您的代理中：

1. **获取规范**：获取 OpenAPI 规范文档（例如从 `.json` 或 `.yaml` 文件加载，或从 URL 获取）。
2. **实例化工具集**：创建 `OpenAPIToolset` 实例，传入规范内容和类型（`spec_str`/`spec_dict`、`spec_str_type`）。如果 API 需要，提供身份验证详情（`auth_scheme`、`auth_credential`）。

    ```python
    from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset

    # 使用 JSON 字符串的示例
    openapi_spec_json = '...' # 您的 OpenAPI JSON 字符串
    toolset = OpenAPIToolset(spec_str=openapi_spec_json, spec_str_type="json")

    # 使用字典的示例
    # openapi_spec_dict = {...} # 您的 OpenAPI 规范字典
    # toolset = OpenAPIToolset(spec_dict=openapi_spec_dict)
    ```

3. **获取工具**：从工具集中获取生成的 `RestApiTool` 实例列表。

    ```python
    api_tools = toolset.get_tools()
    # 或者通过生成的名称（snake_case 格式的 operationId）获取特定工具
    # specific_tool = toolset.get_tool("list_pets")
    ```

4. **添加到代理**：将获取的工具包含在 `LlmAgent` 的 `tools` 列表中。

    ```python
    from google.adk.agents import LlmAgent

    my_agent = LlmAgent(
        name="api_interacting_agent",
        model="gemini-2.0-flash", # 或您偏好的模型
        tools=api_tools, # 传入生成的工具列表
        # ... 其他代理配置 ...
    )
    ```

5. **指导代理**：更新代理的指令，告知其新的 API 功能以及可使用的工具名称（例如 `list_pets`、`create_pet`）。从规范生成的工具描述也将帮助大模型理解。
6. **运行代理**：使用 `Runner` 执行代理。当大模型确定需要调用某个 API 时，它会生成针对相应 `RestApiTool` 的函数调用，后者会自动处理 HTTP 请求。

## 示例

本示例演示了如何从简单的 Pet Store OpenAPI 规范生成工具（使用 `httpbin.org` 模拟响应），并通过代理与之交互。

???+ "代码：Pet Store API"

    ```python title="openapi_example.py"
    --8<-- "examples/python/snippets/tools/openapi_tool.py"
    ```