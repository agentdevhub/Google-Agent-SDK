# 模型上下文协议工具指南

本指南将介绍两种将模型上下文协议（MCP）与ADK集成的方法。

## 什么是模型上下文协议（MCP）？

模型上下文协议（MCP）是一个开放标准，旨在规范Gemini、Claude等大模型与外部应用程序、数据源和工具的通信方式。可以将其视为一种通用连接机制，简化了大模型获取上下文、执行操作以及与各类系统交互的过程。

MCP采用客户端-服务器架构，定义了**数据**（资源）、**交互模板**（提示词）和**可执行功能**（工具）如何通过**MCP服务器**暴露，并由**MCP客户端**（可能是大模型宿主应用或AI代理）消费。

本指南涵盖两种主要集成模式：

1. **在ADK中使用现有MCP服务器**：ADK代理作为MCP客户端，利用外部MCP服务器提供的工具  
2. **通过MCP服务器暴露ADK工具**：构建封装ADK工具的MCP服务器，使其可供任何MCP客户端访问

## 前提条件

开始前请确保已完成以下设置：

* **配置ADK环境**：按照快速入门中的标准ADK[安装]()说明操作  
* **安装/更新Python**：MCP需要Python 3.9或更高版本  
* **安装Node.js和npx**：许多社区MCP服务器以Node.js包形式分发，需使用`npx`运行。若未安装请先安装Node.js（包含npx）。详情参见[https://nodejs.org/en](https://nodejs.org/en)  
* **验证安装**：在激活的虚拟环境中确认`adk`和`npx`已加入PATH：

```shell
# Both commands should print the path to the executables.
which adk
which npx
```

## 1. **ADK代理使用MCP服务器（ADK作为MCP客户端）**

本节展示两个ADK代理使用MCP服务器的示例。这是最常见的集成模式，您的ADK代理需要使用现有服务通过MCP服务器暴露的功能。

### `MCPToolset`类

示例使用ADK中的`MCPToolset`类作为与MCP服务器的桥梁。ADK代理通过`MCPToolset`实现：

1. **连接**：建立与MCP服务器进程的连接（通过标准输入/输出`StdioServerParameters`或使用Server-Sent Events`SseServerParams`的远程服务器）  
2. **发现**：查询MCP服务器获取可用工具（`list_tools` MCP方法）  
3. **适配**：将MCP工具模式转换为ADK兼容的`BaseTool`实例  
4. **暴露**：向ADK `LlmAgent`提供这些适配后的工具  
5. **代理调用**：当`LlmAgent`决定使用某个工具时，`MCPToolset`将调用（`call_tool` MCP方法）转发至MCP服务器并返回结果  
6. **管理连接**：处理与MCP服务器进程的连接生命周期，通常需要显式清理

### 示例1：文件系统MCP服务器

本示例演示连接提供文件系统操作的本地MCP服务器。

#### 步骤1：通过`MCPToolset`将MCP服务器附加到ADK代理

在`./adk_agent_samples/mcp_agent/`中创建`agent.py`，使用以下代码片段初始化`MCPToolset`

* **重要**：将`"/path/to/your/folder"`替换为您系统上的**绝对路径**

```py
# ./adk_agent_samples/mcp_agent/agent.py
import asyncio
from dotenv import load_dotenv
from google.genai import types
from google.adk.agents.llm_agent import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.artifacts.in_memory_artifact_service import InMemoryArtifactService # Optional
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, SseServerParams, StdioServerParameters

# Load environment variables from .env file in the parent directory
# Place this near the top, before using env vars like API keys
load_dotenv('../.env')

# --- Step 1: Import Tools from MCP Server ---
async def get_tools_async():
  """Gets tools from the File System MCP Server."""
  print("Attempting to connect to MCP Filesystem server...")
  tools, exit_stack = await MCPToolset.from_server(
      # Use StdioServerParameters for local process communication
      connection_params=StdioServerParameters(
          command='npx', # Command to run the server
          args=["-y",    # Arguments for the command
                "@modelcontextprotocol/server-filesystem",
                # TODO: IMPORTANT! Change the path below to an ABSOLUTE path on your system.
                "/path/to/your/folder"],
      )
      # For remote servers, you would use SseServerParams instead:
      # connection_params=SseServerParams(url="http://remote-server:port/path", headers={...})
  )
  print("MCP Toolset created successfully.")
  # MCP requires maintaining a connection to the local MCP Server.
  # exit_stack manages the cleanup of this connection.
  return tools, exit_stack

# --- Step 2: Agent Definition ---
async def get_agent_async():
  """Creates an ADK Agent equipped with tools from the MCP Server."""
  tools, exit_stack = await get_tools_async()
  print(f"Fetched {len(tools)} tools from MCP server.")
  root_agent = LlmAgent(
      model='gemini-2.0-flash', # Adjust model name if needed based on availability
      name='filesystem_assistant',
      instruction='Help user interact with the local filesystem using available tools.',
      tools=tools, # Provide the MCP tools to the ADK agent
  )
  return root_agent, exit_stack

# --- Step 3: Main Execution Logic ---
async def async_main():
  session_service = InMemorySessionService()
  # Artifact service might not be needed for this example
  artifacts_service = InMemoryArtifactService()

  session = session_service.create_session(
      state={}, app_name='mcp_filesystem_app', user_id='user_fs'
  )

  # TODO: Change the query to be relevant to YOUR specified folder.
  # e.g., "list files in the 'documents' subfolder" or "read the file 'notes.txt'"
  query = "list files in the tests folder"
  print(f"User Query: '{query}'")
  content = types.Content(role='user', parts=[types.Part(text=query)])

  root_agent, exit_stack = await get_agent_async()

  runner = Runner(
      app_name='mcp_filesystem_app',
      agent=root_agent,
      artifact_service=artifacts_service, # Optional
      session_service=session_service,
  )

  print("Running agent...")
  events_async = runner.run_async(
      session_id=session.id, user_id=session.user_id, new_message=content
  )

  async for event in events_async:
    print(f"Event received: {event}")

  # Crucial Cleanup: Ensure the MCP server process connection is closed.
  print("Closing MCP server connection...")
  await exit_stack.aclose()
  print("Cleanup complete.")

if __name__ == '__main__':
  try:
    asyncio.run(async_main())
  except Exception as e:
    print(f"An error occurred: {e}")
```

#### 步骤2：观察结果

从adk_agent_samples目录运行脚本（确保虚拟环境已激活）：

```shell
cd ./adk_agent_samples
python3 ./mcp_agent/agent.py
```

以下显示连接尝试的预期输出：MCP服务器启动（通过npx）、ADK代理事件（包括对list_directory的FunctionCall和FunctionResponse）以及基于文件列表的最终代理文本响应。确保最后执行exit_stack.aclose()。

```text
User Query: 'list files in the tests folder'
Attempting to connect to MCP Filesystem server...
# --> npx process starts here, potentially logging to stderr/stdout
Secure MCP Filesystem Server running on stdio
Allowed directories: [
  '/path/to/your/folder'
]
# <-- npx process output ends
MCP Toolset created successfully.
Fetched [N] tools from MCP server. # N = number of tools like list_directory, read_file etc.
Running agent...
Event received: content=Content(parts=[Part(..., function_call=FunctionCall(id='...', args={'path': 'tests'}, name='list_directory'), ...)], role='model') ...
Event received: content=Content(parts=[Part(..., function_response=FunctionResponse(id='...', name='list_directory', response={'result': CallToolResult(..., content=[TextContent(...)], ...)}), ...)], role='user') ...
Event received: content=Content(parts=[Part(..., text='https://developers.google.com/maps/get-started#enable-api-sdk')], role='model') ...
Closing MCP server connection...
Cleanup complete.

```

### 示例2：Google地图MCP服务器

遵循相同模式，但针对Google地图MCP服务器。

#### 步骤1：获取API密钥并启用API

按照[使用API密钥](https://developers.google.com/maps/documentation/javascript/get-api-key#create-api-keys)指南获取Google地图API密钥。

在Google Cloud项目中启用Directions API和Routes API。操作说明参见[Google Maps Platform入门](https://developers.google.com/maps/get-started#enable-api-sdk)主题。

#### 步骤2：更新get_tools_async

修改agent.py中的get_tools_async以连接地图服务器，通过StdioServerParameters的env参数传递API密钥。

```py
# agent.py (modify get_tools_async and other parts as needed)
import asyncio
from dotenv import load_dotenv
from google.genai import types
from google.adk.agents.llm_agent import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.artifacts.in_memory_artifact_service import InMemoryArtifactService # Optional
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, SseServerParams, StdioServerParameters

load_dotenv('../.env')

async def get_tools_async():
  """ Step 1: Gets tools from the Google Maps MCP Server."""
  # IMPORTANT: Replace with your actual key
  google_maps_api_key = "YOUR_API_KEY_FROM_STEP_1"
  if "YOUR_API_KEY" in google_maps_api_key:
      raise ValueError("Please replace 'YOUR_API_KEY_FROM_STEP_1' with your actual Google Maps API key.")

  print("Attempting to connect to MCP Google Maps server...")
  tools, exit_stack = await MCPToolset.from_server(
      connection_params=StdioServerParameters(
          command='npx',
          args=["-y",
                "@modelcontextprotocol/server-google-maps",
          ],
          # Pass the API key as an environment variable to the npx process
          env={
              "GOOGLE_MAPS_API_KEY": google_maps_api_key
          }
      )
  )
  print("MCP Toolset created successfully.")
  return tools, exit_stack

# --- Step 2: Agent Definition ---
async def get_agent_async():
  """Creates an ADK Agent equipped with tools from the MCP Server."""
  tools, exit_stack = await get_tools_async()
  print(f"Fetched {len(tools)} tools from MCP server.")
  root_agent = LlmAgent(
      model='gemini-2.0-flash', # Adjust if needed
      name='maps_assistant',
      instruction='Help user with mapping and directions using available tools.',
      tools=tools,
  )
  return root_agent, exit_stack

# --- Step 3: Main Execution Logic (modify query) ---
async def async_main():
  session_service = InMemorySessionService()
  artifacts_service = InMemoryArtifactService() # Optional

  session = session_service.create_session(
      state={}, app_name='mcp_maps_app', user_id='user_maps'
  )

  # TODO: Use specific addresses for reliable results with this server
  query = "What is the route from 1600 Amphitheatre Pkwy to 1165 Borregas Ave"
  print(f"User Query: '{query}'")
  content = types.Content(role='user', parts=[types.Part(text=query)])

  root_agent, exit_stack = await get_agent_async()

  runner = Runner(
      app_name='mcp_maps_app',
      agent=root_agent,
      artifact_service=artifacts_service, # Optional
      session_service=session_service,
  )

  print("Running agent...")
  events_async = runner.run_async(
      session_id=session.id, user_id=session.user_id, new_message=content
  )

  async for event in events_async:
    print(f"Event received: {event}")

  print("Closing MCP server connection...")
  await exit_stack.aclose()
  print("Cleanup complete.")

if __name__ == '__main__':
  try:
    asyncio.run(async_main())
  except Exception as e:
      print(f"An error occurred: {e}")

```

#### 步骤3：观察结果

从adk_agent_samples目录运行脚本（确保虚拟环境已激活）：

```shell
cd ./adk_agent_samples
python3 ./mcp_agent/agent.py
```

成功运行将显示事件日志，表明代理调用了相关Google地图工具（可能与路线规划相关）以及包含路线信息的最终响应。示例如下：

```text
User Query: 'What is the route from 1600 Amphitheatre Pkwy to 1165 Borregas Ave'
Attempting to connect to MCP Google Maps server...
# --> npx process starts...
MCP Toolset created successfully.
Fetched [N] tools from MCP server.
Running agent...
Event received: content=Content(parts=[Part(..., function_call=FunctionCall(name='get_directions', ...))], role='model') ...
Event received: content=Content(parts=[Part(..., function_response=FunctionResponse(name='get_directions', ...))], role='user') ...
Event received: content=Content(parts=[Part(..., text='Head north toward Amphitheatre Pkwy...')], role='model') ...
Closing MCP server connection...
Cleanup complete.

```

## 2. **构建包含ADK工具的MCP服务器（通过MCP暴露ADK）**

此模式允许您封装ADK工具，使其可供任何标准MCP客户端应用使用。本节示例通过MCP服务器暴露load_web_page ADK工具。

### 步骤概览

您将使用model-context-protocol库创建标准Python MCP服务器应用。在该服务器中需要：

1. 实例化要暴露的ADK工具（如FunctionTool(load_web_page)）  
2. 实现MCP服务器的@app.list_tools处理程序来发布ADK工具，使用adk_to_mcp_tool_type将ADK工具定义转换为MCP模式  
3. 实现MCP服务器的@app.call_tool处理程序接收MCP客户端请求，识别是否针对封装的ADK工具，执行ADK工具的.run_async()方法，并将结果格式化为MCP兼容响应（如types.TextContent）

### 前提条件

在ADK相同环境中安装MCP服务器库：

```shell
pip install mcp
```

### 步骤1：创建MCP服务器脚本

新建Python文件，如adk_mcp_server.py。

### 步骤2：实现服务器逻辑

添加以下代码设置暴露ADK load_web_page工具的MCP服务器。

```py
# adk_mcp_server.py
import asyncio
import json
from dotenv import load_dotenv

# MCP Server Imports
from mcp import types as mcp_types # Use alias to avoid conflict with genai.types
from mcp.server.lowlevel import Server, NotificationOptions
from mcp.server.models import InitializationOptions
import mcp.server.stdio

# ADK Tool Imports
from google.adk.tools.function_tool import FunctionTool
from google.adk.tools.load_web_page import load_web_page # Example ADK tool
# ADK <-> MCP Conversion Utility
from google.adk.tools.mcp_tool.conversion_utils import adk_to_mcp_tool_type

# --- Load Environment Variables (If ADK tools need them) ---
load_dotenv()

# --- Prepare the ADK Tool ---
# Instantiate the ADK tool you want to expose
print("Initializing ADK load_web_page tool...")
adk_web_tool = FunctionTool(load_web_page)
print(f"ADK tool '{adk_web_tool.name}' initialized.")
# --- End ADK Tool Prep ---

# --- MCP Server Setup ---
print("Creating MCP Server instance...")
# Create a named MCP Server instance
app = Server("adk-web-tool-mcp-server")

# Implement the MCP server's @app.list_tools handler
@app.list_tools()
async def list_tools() -> list[mcp_types.Tool]:
  """MCP handler to list available tools."""
  print("MCP Server: Received list_tools request.")
  # Convert the ADK tool's definition to MCP format
  mcp_tool_schema = adk_to_mcp_tool_type(adk_web_tool)
  print(f"MCP Server: Advertising tool: {mcp_tool_schema.name}")
  return [mcp_tool_schema]

# Implement the MCP server's @app.call_tool handler
@app.call_tool()
async def call_tool(
    name: str, arguments: dict
) -> list[mcp_types.TextContent | mcp_types.ImageContent | mcp_types.EmbeddedResource]:
  """MCP handler to execute a tool call."""
  print(f"MCP Server: Received call_tool request for '{name}' with args: {arguments}")

  # Check if the requested tool name matches our wrapped ADK tool
  if name == adk_web_tool.name:
    try:
      # Execute the ADK tool's run_async method
      # Note: tool_context is None as we are not within a full ADK Runner invocation
      adk_response = await adk_web_tool.run_async(
          args=arguments,
          tool_context=None, # No ADK context available here
      )
      print(f"MCP Server: ADK tool '{name}' executed successfully.")
      # Format the ADK tool's response (often a dict) into MCP format.
      # Here, we serialize the response dictionary as a JSON string within TextContent.
      # Adjust formatting based on the specific ADK tool's output and client needs.
      response_text = json.dumps(adk_response, indent=2)
      return [mcp_types.TextContent(type="text", text=response_text)]

    except Exception as e:
      print(f"MCP Server: Error executing ADK tool '{name}': {e}")
      # Return an error message in MCP format
      # Creating a proper MCP error response might be more robust
      error_text = json.dumps({"error": f"Failed to execute tool '{name}': {str(e)}"})
      return [mcp_types.TextContent(type="text", text=error_text)]
  else:
      # Handle calls to unknown tools
      print(f"MCP Server: Tool '{name}' not found.")
      error_text = json.dumps({"error": f"Tool '{name}' not implemented."})
      # Returning error as TextContent for simplicity
      return [mcp_types.TextContent(type="text", text=error_text)]

# --- MCP Server Runner ---
async def run_server():
  """Runs the MCP server over standard input/output."""
  # Use the stdio_server context manager from the MCP library
  async with mcp.server.stdio.stdio_server() as (read_stream, write_stream):
    print("MCP Server starting handshake...")
    await app.run(
        read_stream,
        write_stream,
        InitializationOptions(
            server_name=app.name, # Use the server name defined above
            server_version="0.1.0",
            capabilities=app.get_capabilities(
                # Define server capabilities - consult MCP docs for options
                notification_options=NotificationOptions(),
                experimental_capabilities={},
            ),
        ),
    )
    print("MCP Server run loop finished.")

if __name__ == "__main__":
  print("Launching MCP Server exposing ADK tools...")
  try:
    asyncio.run(run_server())
  except KeyboardInterrupt:
    print("\nMCP Server stopped by user.")
  except Exception as e:
    print(f"MCP Server encountered an error: {e}")
  finally:
    print("MCP Server process exiting.")
# --- End MCP Server ---

```

### 步骤3：使用ADK测试MCP服务器

按照"示例1：文件系统MCP服务器"相同说明创建MCP客户端。这次使用上述创建的MCP服务器文件作为输入命令：

```py
# ./adk_agent_samples/mcp_agent/agent.py

# ...

async def get_tools_async():
  """Gets tools from the File System MCP Server."""
  print("Attempting to connect to MCP Filesystem server...")
  tools, exit_stack = await MCPToolset.from_server(
      # Use StdioServerParameters for local process communication
      connection_params=StdioServerParameters(
          command='python3', # Command to run the server
          args=[
                "/absolute/path/to/adk_mcp_server.py"],
      )
  )
```

从终端执行代理脚本（确保环境中已安装model-context-protocol和google-adk等必要库）：

```shell
cd ./adk_agent_samples
python3 ./mcp_agent/agent.py
```

脚本将打印启动消息，然后等待MCP客户端通过标准输入/输出连接到adk_mcp_server.py中的MCP服务器。任何MCP兼容客户端（如Claude Desktop或使用MCP库的自定义客户端）现在都可以连接此进程，发现load_web_page工具并调用它。服务器将打印日志消息显示接收到的请求和ADK工具执行情况。参考[文档](https://modelcontextprotocol.io/quickstart/server#core-mcp-concepts)尝试与Claude Desktop集成。

## 关键注意事项

使用MCP和ADK时需注意：

* **协议与库的区别**：MCP是协议规范，定义通信规则；ADK是构建代理的Python库/框架。MCPToolset通过在ADK框架内实现MCP协议的客户端来桥接二者。反之，构建Python MCP服务器需要使用model-context-protocol库。

* **ADK工具与MCP工具的区别**：
    * ADK工具（BaseTool、FunctionTool、AgentTool等）是为ADK的LlmAgent和Runner直接使用的Python对象  
    * MCP工具是MCP服务器根据协议模式暴露的能力。MCPToolset使这些工具对LlmAgent表现为ADK工具  
    * Langchain/CrewAI工具是这些库中的特定实现，通常是简单函数或类，缺乏MCP的服务器/协议结构。ADK提供部分互操作封装器（LangchainTool、CrewaiTool）

* **异步特性**：ADK和MCP Python库都深度依赖asyncio库。工具实现和服务器处理程序通常应为异步函数。

* **有状态会话（MCP）**：MCP在客户端和服务器实例间建立有状态的持久连接，这与典型的无状态REST API不同。
    * **部署**：这种有状态性可能带来扩展和部署挑战，特别是远程服务器处理多用户时。原始MCP设计通常假设客户端和服务器共置。管理这些持久连接需要谨慎的基础设施考虑（如负载均衡、会话亲和性）  
    * **ADK MCPToolset**：管理此连接生命周期。示例中展示的exit_stack模式对确保连接（以及可能的服务器进程）在ADK代理结束时正确终止至关重要。

## 扩展资源

* [模型上下文协议文档](https://modelcontextprotocol.io/)  
* [MCP规范](https://modelcontextprotocol.io/specification/)  
* [MCP Python SDK及示例](https://github.com/modelcontextprotocol/)