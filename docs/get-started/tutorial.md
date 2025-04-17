# 构建你的第一个智能体团队：基于ADK的渐进式天气机器人

<!-- 可选的外层容器，用于整体内边距/间距控制 -->
  <div style="margin-bottom: 10px;">
    <a href="https://colab.research.google.com/github/google/adk-docs/blob/main/examples/python/notebooks/adk_tutorial.ipynb" target="_blank" style="display: inline-flex; align-items: center; gap: 5px; text-decoration: none; color: #4285F4;">
      <img width="32px" src="https://www.gstatic.com/pantheon/images/bigquery/welcome_page/colab-logo.svg" alt="Google Colaboratory logo">
      <span>Open in Colab</span>
    </a>
  </div>

  <!-- 第二行：分享链接 -->
  <!-- 这个div作为"分享到"文本和图标的弹性容器 -->
  <div style="display: flex; align-items: center; gap: 10px; flex-wrap: wrap;">
    <!-- Share Text -->
    <span style="font-weight: bold;">Share to:</span>

    <!-- Social Media Links -->
    <a href="https://www.linkedin.com/sharing/share-offsite/?url=https%3A//github.com/google/adk-docs/blob/main/examples/python/notebooks/adk_tutorial.ipynb" target="_blank" title="Share on LinkedIn">
      <img width="20px" src="https://upload.wikimedia.org/wikipedia/commons/8/81/LinkedIn_icon.svg" alt="LinkedIn logo" style="vertical-align: middle;">
    </a>
    <a href="https://bsky.app/intent/compose?text=https%3A//github.com/google/adk-docs/blob/main/examples/python/notebooks/adk_tutorial.ipynb" target="_blank" title="分享至Bluesky">
      <img width="20px" src="https://upload.wikimedia.org/wikipedia/commons/7/7a/Bluesky_Logo.svg" alt="Bluesky logo" style="vertical-align: middle;">
    </a>
    <a href="https://twitter.com/intent/tweet?url=https%3A//github.com/google/adk-docs/blob/main/examples/python/notebooks/adk_tutorial.ipynb" target="_blank" title="分享至X（推特）">
      <img width="20px" src="https://upload.wikimedia.org/wikipedia/commons/5/5a/X_icon_2.svg" alt="X logo" style="vertical-align: middle;">
    </a>
    <a href="https://reddit.com/submit?url=https%3A//github.com/google/adk-docs/blob/main/examples/python/notebooks/adk_tutorial.ipynb" target="_blank" title="分享至Reddit">
      <img width="20px" src="https://redditinc.com/hubfs/Reddit%20Inc/Brand/Reddit_Logo.png" alt="Reddit logo" style="vertical-align: middle;">
    </a>
    <a href="https://www.facebook.com/sharer/sharer.php?u=https%3A//github.com/google/adk-docs/blob/main/examples/python/notebooks/adk_tutorial.ipynb" target="_blank" title="分享至Facebook">
      <img width="20px" src="https://upload.wikimedia.org/wikipedia/commons/5/51/Facebook_f_logo_%282019%29.svg" alt="Facebook logo" style="vertical-align: middle;">
    </a>
  </div>

</div>
本教程基于[Agent Development Kit](https://google.github.io/adk-docs/get-started/)的[快速入门示例](https://google.github.io/adk-docs/get-started/quickstart/)扩展而来。现在，你已经准备好深入探索并构建一个更复杂的**多智能体系统**了。

我们将着手构建一个**天气机器人智能体团队**，在简单基础上逐步叠加高级功能。从一个能查询天气的单一智能体开始，我们将逐步添加以下能力：

* 利用不同AI模型（Gemini、GPT、Claude）  
* 为不同任务设计专用子智能体（如问候和告别）  
* 实现智能体间的智能委派  
* 通过持久会话状态赋予智能体记忆能力  
* 使用回调函数实现关键安全防护机制  

**为什么选择天气机器人团队？**

这个看似简单的用例，提供了一个实用且易于理解的场景，来探索构建复杂现实世界智能应用所需的核心ADK概念。你将学习如何构建交互结构、管理状态、确保安全性，以及协调多个AI"大脑"协同工作。

**再认识一下ADK**

ADK是一个Python框架，旨在简化基于大模型(LLMs)的应用开发。它提供了强大的构建模块，用于创建能够推理、规划、使用工具、与用户动态交互，并在团队中有效协作的智能体。

**在本进阶教程中，你将掌握：**

* ✅ **工具定义与使用**：编写Python函数(`tools`)，赋予智能体特定能力（如获取数据），并指导智能体如何有效使用它们  
* ✅ **多模型灵活性**：通过LiteLLM集成配置智能体使用各种主流大模型（Gemini、GPT-4o、Claude Sonnet），让你能为每项任务选择最佳模型  
* ✅ **智能体委派与协作**：设计专用子智能体，实现用户请求在团队内自动路由(`auto flow`)到最合适的智能体  
* ✅ **会话状态记忆**：使用`Session State`和`ToolContext`让智能体记住跨对话轮次的信息，实现更具上下文感知的交互  
* ✅ **回调安全防护**：实现`before_model_callback`和`before_tool_callback`来检查、修改或基于预定义规则阻止请求/工具使用，增强应用安全性和控制力  

**最终成果预期：**

完成本教程后，你将构建一个功能完善的多智能体天气机器人系统。该系统不仅能提供天气信息，还能处理礼貌对话、记住上次查询的城市，并在定义的安全边界内运行，所有功能都通过ADK协调实现。

**先决条件：**

* ✅ **扎实的Python编程基础**  
* ✅ **熟悉大模型(LLMs)、API和智能体概念**  
* ❗ **关键要求：完成ADK快速入门教程或具备同等ADK基础知识（Agent、Runner、SessionService、基本工具使用）**。本教程直接基于这些概念构建  
* ✅ **计划使用的大模型API密钥**（如Google AI Studio的Gemini、OpenAI Platform、Anthropic Console）

**准备好构建你的智能体团队了吗？让我们开始吧！**

## 第0步：设置与安装

### 库安装

```

!pip install google-adk -q
!pip install litellm -q

print("Installation complete.")
```

### 导入库

```
import os
import asyncio
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm # For multi-model support
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai import types # For creating message Content/Parts

import warnings
# Ignore all warnings
warnings.filterwarnings("ignore")

import logging
logging.basicConfig(level=logging.ERROR)

print("Libraries imported.")
```

### 设置API密钥

```

# --- IMPORTANT: Replace placeholders with your real API keys ---

# Gemini API Key (Get from Google AI Studio: https://aistudio.google.com/app/apikey)
os.environ["GOOGLE_API_KEY"] = "YOUR_GOOGLE_API_KEY" # <--- REPLACE

# OpenAI API Key (Get from OpenAI Platform: https://platform.openai.com/api-keys)
os.environ['OPENAI_API_KEY'] = 'YOUR_OPENAI_API_KEY' # <--- REPLACE

# Anthropic API Key (Get from Anthropic Console: https://console.anthropic.com/settings/keys)
os.environ['ANTHROPIC_API_KEY'] = 'YOUR_ANTHROPIC_API_KEY' # <--- REPLACE


# --- Verify Keys (Optional Check) ---
print("API Keys Set:")
print(f"Google API Key set: {'Yes' if os.environ.get('GOOGLE_API_KEY') and os.environ['GOOGLE_API_KEY'] != 'YOUR_GOOGLE_API_KEY' else 'No (REPLACE PLACEHOLDER!)'}")
print(f"OpenAI API Key set: {'Yes' if os.environ.get('OPENAI_API_KEY') and os.environ['OPENAI_API_KEY'] != 'YOUR_OPENAI_API_KEY' else 'No (REPLACE PLACEHOLDER!)'}")
print(f"Anthropic API Key set: {'Yes' if os.environ.get('ANTHROPIC_API_KEY') and os.environ['ANTHROPIC_API_KEY'] != 'YOUR_ANTHROPIC_API_KEY' else 'No (REPLACE PLACEHOLDER!)'}")

# Configure ADK to use API keys directly (not Vertex AI for this multi-model setup)
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "False"


# @markdown **Security Note:** It's best practice to manage API keys securely (e.g., using Colab Secrets or environment variables) rather than hardcoding them directly in the notebook. Replace the placeholder strings above.

```

### 定义模型常量以便使用

```
MODEL_GEMINI_2_0_FLASH = "gemini-2.0-flash"

# Note: Specific model names might change. Refer to LiteLLM or the model provider's documentation.
MODEL_GPT_4O = "openai/gpt-4o"
MODEL_CLAUDE_SONNET = "anthropic/claude-3-sonnet-20240229"


print("\nEnvironment configured.")
```

## 第1步：你的第一个智能体 - 基础天气查询

让我们从构建天气机器人的基础组件开始：一个能执行特定任务（查询天气信息）的单一智能体。这需要创建两个核心部分：

1. **工具**：一个Python函数，赋予智能体获取天气数据的*能力*  
2. **智能体**：理解用户请求、知道自己有天气工具，并决定何时及如何使用该工具的AI"大脑"

---

### **1. 定义工具**

在ADK中，**工具**是赋予智能体超越文本生成的具体能力的构建块。它们通常是执行特定操作的常规Python函数，如调用API、查询数据库或执行计算。

我们的第一个工具将提供*模拟*天气报告。这让我们可以专注于智能体结构，而无需外部API密钥。之后，你可以轻松将这个模拟函数替换为调用真实天气服务的函数。

**关键概念：文档字符串至关重要！** 智能体的大模型严重依赖函数的**文档字符串**来理解：

* 工具*做什么*  
* *何时*使用它  
* 它需要*什么参数*(`city: str`)  
* 它返回*什么信息*

**最佳实践**：为工具编写清晰、描述性强且准确的文档字符串。这对大模型正确使用工具至关重要。

```py
# @title Define the get_weather Tool
def get_weather(city: str) -> dict:
    """Retrieves the current weather report for a specified city.

    Args:
        city (str): The name of the city (e.g., "New York", "London", "Tokyo").

    Returns:
        dict: A dictionary containing the weather information.
              Includes a 'status' key ('success' or 'error').
              If 'success', includes a 'report' key with weather details.
              If 'error', includes an 'error_message' key.
    """
    # Best Practice: Log tool execution for easier debugging
    print(f"--- Tool: get_weather called for city: {city} ---")
    city_normalized = city.lower().replace(" ", "") # Basic input normalization

    # Mock weather data for simplicity
    mock_weather_db = {
        "newyork": {"status": "success", "report": "The weather in New York is sunny with a temperature of 25°C."},
        "london": {"status": "success", "report": "It's cloudy in London with a temperature of 15°C."},
        "tokyo": {"status": "success", "report": "Tokyo is experiencing light rain and a temperature of 18°C."},
    }

    # Best Practice: Handle potential errors gracefully within the tool
    if city_normalized in mock_weather_db:
        return mock_weather_db[city_normalized]
    else:
        return {"status": "error", "error_message": f"Sorry, I don't have weather information for '{city}'."}

# Example tool usage (optional self-test)
print(get_weather("New York"))
print(get_weather("Paris"))
```

---

### **2. 定义智能体**

现在，让我们创建**智能体**本身。ADK中的`Agent`协调用户、大模型和可用工具之间的交互。

我们通过几个关键参数配置它：

* `name`：该智能体的唯一标识符（如"weather_agent_v1"）  
* `model`：指定使用哪个大模型（如`MODEL_GEMINI_2_5_PRO`）。我们从特定的Gemini模型开始  
* `description`：智能体整体用途的简明摘要。这在其他智能体需要决定是否将任务委派给*此*智能体时变得至关重要  
* `instruction`：详细指导大模型如何行为、其角色、目标，特别是*如何及何时*使用其分配的`tools`  
* `tools`：包含智能体允许使用的实际Python工具函数的列表（如`[get_weather]`）

**最佳实践**：提供清晰具体的`instruction`提示。指令越详细，大模型越能理解其角色和如何有效使用工具。如果需要，明确说明错误处理。

**最佳实践**：选择描述性的`name`和`description`值。这些由ADK内部使用，对自动委派等功能至关重要（稍后介绍）。

```py
# @title Define the Weather Agent
# Use one of the model constants defined earlier
AGENT_MODEL = MODEL_GEMINI_2_5_PRO # Starting with a powerful Gemini model

weather_agent = Agent(
    name="weather_agent_v1",
    model=AGENT_MODEL, # Specifies the underlying LLM
    description="Provides weather information for specific cities.", # Crucial for delegation later
    instruction="You are a helpful weather assistant. Your primary goal is to provide current weather reports. "
                "When the user asks for the weather in a specific city, "
                "you MUST use the 'get_weather' tool to find the information. "
                "Analyze the tool's response: if the status is 'error', inform the user politely about the error message. "
                "If the status is 'success', present the weather 'report' clearly and concisely to the user. "
                "Only use the tool when a city is mentioned for a weather request.",
    tools=[get_weather], # Make the tool available to this agent
)

print(f"Agent '{weather_agent.name}' created using model '{AGENT_MODEL}'.")
```

---

### **3. 设置Runner和Session Service**

为了管理对话和执行智能体，我们还需要两个组件：

* `SessionService`：负责管理不同用户和会话的对话历史和状态。`InMemorySessionService`是一个简单实现，将所有内容存储在内存中，适合测试和简单应用。它跟踪交换的消息。我们将在第4步进一步探索状态持久化  
* `Runner`：协调交互流程的引擎。它接收用户输入，将其路由到适当的智能体，根据智能体逻辑管理对大模型和工具的调用，通过`SessionService`处理会话更新，并生成表示交互进度的事件

```py
# @title Setup Session Service and Runner

# --- Session Management ---
# Key Concept: SessionService stores conversation history & state.
# InMemorySessionService is simple, non-persistent storage for this tutorial.
session_service = InMemorySessionService()

# Define constants for identifying the interaction context
APP_NAME = "weather_tutorial_app"
USER_ID = "user_1"
SESSION_ID = "session_001" # Using a fixed ID for simplicity

# Create the specific session where the conversation will happen
session = session_service.create_session(
    app_name=APP_NAME,
    user_id=USER_ID,
    session_id=SESSION_ID
)
print(f"Session created: App='{APP_NAME}', User='{USER_ID}', Session='{SESSION_ID}'")

# --- Runner ---
# Key Concept: Runner orchestrates the agent execution loop.
runner = Runner(
    agent=weather_agent, # The agent we want to run
    app_name=APP_NAME,   # Associates runs with our app
    session_service=session_service # Uses our session manager
)
print(f"Runner created for agent '{runner.agent.name}'.")
```

---

### **4. 与智能体交互**

我们需要一种向智能体发送消息并接收其响应的方法。由于大模型调用和工具执行可能需要时间，ADK的`Runner`异步运行。

我们将定义一个`async`辅助函数(`call_agent_async`)，该函数：

1. 接收用户查询字符串  
2. 将其打包为ADK `Content`格式  
3. 调用`runner.run_async`，提供用户/会话上下文和新消息  
4. 遍历runner生成的**事件**。事件表示智能体执行中的步骤（如请求工具调用、接收工具结果、中间大模型思考、最终响应）  
5. 使用`event.is_final_response()`识别并打印**最终响应**事件

**为什么使用`async`？** 与大模型和潜在工具（如外部API）的交互是I/O密集型操作。使用`asyncio`允许程序高效处理这些操作而不会阻塞执行。

```py
# @title Define Agent Interaction Function
import asyncio
from google.genai import types # For creating message Content/Parts

async def call_agent_async(query: str):
  """Sends a query to the agent and prints the final response."""
  print(f"\n>>> User Query: {query}")

  # Prepare the user's message in ADK format
  content = types.Content(role='user', parts=[types.Part(text=query)])

  final_response_text = "Agent did not produce a final response." # Default

  # Key Concept: run_async executes the agent logic and yields Events.
  # We iterate through events to find the final answer.
  async for event in runner.run_async(user_id=USER_ID, session_id=SESSION_ID, new_message=content):
      # You can uncomment the line below to see *all* events during execution
      # print(f"  [Event] Author: {event.author}, Type: {type(event).__name__}, Final: {event.is_final_response()}, Content: {event.content}")

      # Key Concept: is_final_response() marks the concluding message for the turn.
      if event.is_final_response():
          if event.content and event.content.parts:
             # Assuming text response in the first part
             final_response_text = event.content.parts[0].text
          elif event.actions and event.actions.escalate: # Handle potential errors/escalations
             final_response_text = f"Agent escalated: {event.error_message or 'No specific message.'}"
          # Add more checks here if needed (e.g., specific error codes)
          break # Stop processing events once the final response is found

  print(f"<<< Agent Response: {final_response_text}")

```

---

### **5. 运行对话**

最后，让我们通过向智能体发送几个查询来测试我们的设置。我们将`async`调用包装在主要的`async`函数中，并使用`await`运行它。

观察输出：

* 查看用户查询  
* 注意智能体使用工具时的`--- Tool: get_weather called... ---`日志  
* 观察智能体的最终响应，包括它如何处理巴黎天气数据不可用的情况

```py
# @title Run the Initial Conversation

# We need an async function to await our interaction helper
async def run_conversation():
    await call_agent_async("What is the weather like in London?")
    await call_agent_async("How about Paris?") # Expecting the tool's error message
    await call_agent_async("Tell me the weather in New York")

# Execute the conversation using await in an async context (like Colab/Jupyter)
await run_conversation()
```

**预期输出：**

```console
>>> User Query: What is the weather like in London?

--- Tool: get_weather called for city: London ---
<<< Agent Response: The weather in London is cloudy with a temperature of 15°C.


>>> User Query: How about Paris?

--- Tool: get_weather called for city: Paris ---
<<< Agent Response: Sorry, I don't have weather information for Paris.


>>> User Query: Tell me the weather in New York

--- Tool: get_weather called for city: New York ---
<<< Agent Response: The weather in New York is sunny with a temperature of 25°C.
```

---

恭喜！你已成功构建并与你的第一个ADK智能体进行了交互。它能理解用户请求，使用工具查找信息，并根据工具结果做出适当响应。

下一步，我们将探索如何轻松切换支撑该智能体的底层语言模型。

## 第2步：使用LiteLLM实现多模型支持

在第1步中，我们构建了一个由特定Gemini模型驱动的功能性天气智能体。虽然有效，但现实世界的应用通常受益于使用*不同*大模型(LLMs)的灵活性。为什么？

* **性能**：某些模型擅长特定任务（如编码、推理、创意写作）  
* **成本**：不同模型有不同价格点  
* **能力**：模型提供多样化功能、上下文窗口大小和微调选项  
* **可用性/冗余**：拥有备选方案确保即使一个提供商出现问题，你的应用仍能正常运行  

ADK通过与[**LiteLLM**](https://github.com/BerriAI/litellm)库的集成，使模型切换变得无缝。LiteLLM作为100多种不同大模型的一致接口。

**在本步骤中，我们将：**

1. 学习如何配置ADK `Agent`使用OpenAI(GPT)和Anthropic(Claude)等提供商的模型，使用`LiteLlm`包装器  
2. 定义、配置（使用它们自己的会话和runner）并立即测试我们的天气智能体实例，每个实例由不同大模型支持  
3. 与这些不同智能体交互，观察即使使用相同底层工具，它们的响应也可能存在差异  

---

### **1. 导入`LiteLlm`**

我们在初始设置（第0步）中已导入此组件，但它是多模型支持的关键：

```py
# Ensure this import is present from your setup cells
from google.adk.models.lite_llm import LiteLlm
```

---

### **2. 定义并测试多模型智能体**

我们不再仅传递模型名称字符串（默认为Google的Gemini模型），而是将所需的模型标识符字符串包装在`LiteLlm`类中。

* **关键概念：`LiteLlm`包装器**：`LiteLlm(model="provider/model_name")`语法告诉ADK通过LiteLLM库将该智能体的请求路由到指定的模型提供商  

确保你已在第0步配置了OpenAI和Anthropic的必要API密钥。我们将使用之前定义的`call_agent_async`函数（现在接受`runner`、`user_id`和`session_id`）在设置后立即与每个智能体交互。

下面的每个代码块将：
* 使用特定的LiteLLM模型(`MODEL_GPT_4O`或`MODEL_CLAUDE_SONNET`)定义智能体  
* 为该智能体的测试运行创建*新的独立*`InMemorySessionService`和会话。这为本次演示保持对话历史的隔离  
* 创建配置给特定智能体及其会话服务的`Runner`  
* 立即调用`call_agent_async`发送查询并测试智能体  

**最佳实践**：使用常量表示模型名称（如第0步定义的`MODEL_GPT_4O`、`MODEL_CLAUDE_SONNET`），避免拼写错误并使代码更易管理  

**错误处理**：我们将智能体定义包装在`try...except`块中。这防止整个代码单元因特定提供商的API密钥缺失或无效而失败，允许教程继续使用*已配置*的模型进行  

首先，让我们创建并测试使用OpenAI的GPT-4o的智能体。

```py
# @title Define and Test GPT Agent

# Make sure 'get_weather' function from Step 1 is defined in your environment.
# Make sure 'call_agent_async' is defined from earlier.

# --- Agent using GPT-4o ---
weather_agent_gpt = None # Initialize to None
runner_gpt = None      # Initialize runner to None

try:
    weather_agent_gpt = Agent(
        name="weather_agent_gpt",
        # Key change: Wrap the LiteLLM model identifier
        model=LiteLlm(model=MODEL_GPT_4O),
        description="Provides weather information (using GPT-4o).",
        instruction="You are a helpful weather assistant powered by GPT-4o. "
                    "Use the 'get_weather' tool for city weather requests. "
                    "Clearly present successful reports or polite error messages based on the tool's output status.",
        tools=[get_weather], # Re-use the same tool
    )
    print(f"Agent '{weather_agent_gpt.name}' created using model '{MODEL_GPT_4O}'.")

    # InMemorySessionService is simple, non-persistent storage for this tutorial.
    session_service_gpt = InMemorySessionService() # Create a dedicated service

    # Define constants for identifying the interaction context
    APP_NAME_GPT = "weather_tutorial_app_gpt" # Unique app name for this test
    USER_ID_GPT = "user_1_gpt"
    SESSION_ID_GPT = "session_001_gpt" # Using a fixed ID for simplicity

    # Create the specific session where the conversation will happen
    session_gpt = session_service_gpt.create_session(
        app_name=APP_NAME_GPT,
        user_id=USER_ID_GPT,
        session_id=SESSION_ID_GPT
    )
    print(f"Session created: App='{APP_NAME_GPT}', User='{USER_ID_GPT}', Session='{SESSION_ID_GPT}'")

    # Create a runner specific to this agent and its session service
    runner_gpt = Runner(
        agent=weather_agent_gpt,
        app_name=APP_NAME_GPT,       # Use the specific app name
        session_service=session_service_gpt # Use the specific session service
        )
    print(f"Runner created for agent '{runner_gpt.agent.name}'.")

    # --- Test the GPT Agent ---
    print("\n--- Testing GPT Agent ---")
    # Ensure call_agent_async uses the correct runner, user_id, session_id
    await call_agent_async(query = "What's the weather in Tokyo?",
                           runner=runner_gpt,
                           user_id=USER_ID_GPT,
                           session_id=SESSION_ID_GPT)

except Exception as e:
    print(f"❌ Could not create or run GPT agent '{MODEL_GPT_4O}'. Check API Key and model name. Error: {e}")

```

接下来，我们对Anthropic的Claude Sonnet做同样操作。

```py
# @title Define and Test Claude Agent

# Make sure 'get_weather' function from Step 1 is defined in your environment.
# Make sure 'call_agent_async' is defined from earlier.

# --- Agent using Claude Sonnet ---
weather_agent_claude = None # Initialize to None
runner_claude = None      # Initialize runner to None

try:
    weather_agent_claude = Agent(
        name="weather_agent_claude",
        # Key change: Wrap the LiteLLM model identifier
        model=LiteLlm(model=MODEL_CLAUDE_SONNET),
        description="Provides weather information (using Claude Sonnet).",
        instruction="You are a helpful weather assistant powered by Claude Sonnet. "
                    "Use the 'get_weather' tool for city weather requests. "
                    "Analyze the tool's dictionary output ('status', 'report'/'error_message'). "
                    "Clearly present successful reports or polite error messages.",
        tools=[get_weather], # Re-use the same tool
    )
    print(f"Agent '{weather_agent_claude.name}' created using model '{MODEL_CLAUDE_SONNET}'.")

    # InMemorySessionService is simple, non-persistent storage for this tutorial.
    session_service_claude = InMemorySessionService() # Create a dedicated service

    # Define constants for identifying the interaction context
    APP_NAME_CLAUDE = "weather_tutorial_app_claude" # Unique app name
    USER_ID_CLAUDE = "user_1_claude"
    SESSION_ID_CLAUDE = "session_001_claude" # Using a fixed ID for simplicity

    # Create the specific session where the conversation will happen
    session_claude = session_service_claude.create_session(
        app_name=APP_NAME_CLAUDE,
        user_id=USER_ID_CLAUDE,
        session_id=SESSION_ID_CLAUDE
    )
    print(f"Session created: App='{APP_NAME_CLAUDE}', User='{USER_ID_CLAUDE}', Session='{SESSION_ID_CLAUDE}'")

    # Create a runner specific to this agent and its session service
    runner_claude = Runner(
        agent=weather_agent_claude,
        app_name=APP_NAME_CLAUDE,       # Use the specific app name
        session_service=session_service_claude # Use the specific session service
        )
    print(f"Runner created for agent '{runner_claude.agent.name}'.")

    # --- Test the Claude Agent ---
    print("\n--- Testing Claude Agent ---")
    # Ensure call_agent_async uses the correct runner, user_id, session_id
    await call_agent_async(query = "Weather in London please.",
                           runner=runner_claude,
                           user_id=USER_ID_CLAUDE,
                           session_id=SESSION_ID_CLAUDE)

except Exception as e:
    print(f"❌ Could not create or run Claude agent '{MODEL_CLAUDE_SONNET}'. Check API Key and model name. Error: {e}")
```

仔细观察两个代码块的输出。你应该看到：

1. 每个智能体(`weather_agent_gpt`、`weather_agent_claude`)成功创建（如果API密钥有效）  
2. 为每个智能体设置了专用会话和runner  
3. 每个智能体在处理查询时正确识别需要使用`get_weather`工具（你会看到`--- Tool: get_weather called... ---`日志）  
4. *底层工具逻辑*保持相同，始终返回我们的模拟数据  
5. 但每个智能体生成的**最终文本响应**可能在措辞、语气或格式上略有不同。这是因为指令提示由不同大模型(GPT-4o vs Claude Sonnet)解释执行  

这一步展示了ADK + LiteLLM提供的强大功能和灵活性。你可以轻松尝试和部署使用各种大模型的智能体，同时保持核心应用逻辑（工具、基本智能体结构）一致。

下一步，我们将超越单一智能体，构建一个小型团队，让智能体可以相互委派任务！

---

## 第3步：构建智能体团队 - 问候与告别的委派

在第1和2步中，我们构建并实验了一个专注于天气查询的单一智能体。虽然对其特定任务有效，但现实世界的应用通常涉及处理更广泛的用户交互。我们可以继续向单一天气智能体添加更多工具和复杂指令，但这很快就会变得难以管理且效率低下。

更稳健的方法是构建一个**智能体团队**。这涉及：

1. 创建多个**专用智能体**，每个智能体设计用于特定能力（如一个处理天气，一个处理问候，一个处理计算）  
2. 指定一个**根智能体**（或协调器）接收初始用户请求  
3. 使根智能体能够基于用户意图将请求**委派**给最合适的专用子智能体  

**为什么要构建智能体团队？**

* **模块化**：更易于开发、测试和维护单个智能体  
* **专业化**：每个智能体可以针对其特定任务进行微调（指令、模型选择）  
* **可扩展性**：通过添加新智能体更简单地增加新能力  
* **效率**：允许对简单任务（如问候）使用可能更简单/更便宜的模型  

**在本步骤中，我们将：**

1. 定义处理问候(`say_hello`)和告别(`say_goodbye`)的简单工具  
2. 创建两个新的专用子智能体：`greeting_agent`和`farewell_agent`  
3. 更新我们的主天气智能体(`weather_agent_v2`)作为**根智能体**  
4. 用其子智能体配置根智能体，实现**自动委派**  
5. 通过向根智能体发送不同类型的请求测试委派流程  

---

### **1. 为子智能体定义工具**

首先，让我们创建将作为新专家智能体工具的简单Python函数。记住，清晰的文档字符串对将使用它们的智能体至关重要。

```py
# @title Define Tools for Greeting and Farewell Agents

# Ensure 'get_weather' from Step 1 is available if running this step independently.
# def get_weather(city: str) -> dict: ... (from Step 1)

def say_hello(name: str = "there") -> str:
    """Provides a simple greeting, optionally addressing the user by name.

    Args:
        name (str, optional): The name of the person to greet. Defaults to "there".

    Returns:
        str: A friendly greeting message.
    """
    print(f"--- Tool: say_hello called with name: {name} ---")
    return f"Hello, {name}!"

def say_goodbye() -> str:
    """Provides a simple farewell message to conclude the conversation."""
    print(f"--- Tool: say_goodbye called ---")
    return "Goodbye! Have a great day."

print("Greeting and Farewell tools defined.")

# Optional self-test
print(say_hello("Alice"))
print(say_goodbye())
```

---

### **2. 定义子智能体（问候与告别）**

现在，为我们的专家创建`Agent`实例。注意它们高度聚焦的`instruction`，以及关键的清晰`description`。`description`是*根智能体*用来决定*何时*委派给这些子智能体的主要信息。

我们甚至可以为这些子智能体使用不同的大模型！让我们为问候智能体分配GPT-4o，并保持告别智能体也使用GPT-4o（如果愿意且API密钥已设置，你可以轻松切换其中一个使用Claude或Gemini）。

**最佳实践**：子智能体的`description`字段应准确简洁地总结其特定能力。这对有效的自动委派至关重要。

**最佳实践**：子智能体的`instruction`字段应针对其有限范围定制，明确告诉它们该做什么和*不该*做什么（如"你的*唯一*任务是..."）。

```py
# @title Define Greeting and Farewell Sub-Agents

# Ensure LiteLlm is imported and API keys are set (from Step 0/2)
# from google.adk.models.lite_llm import LiteLlm
# MODEL_GPT_4O, MODEL_CLAUDE_SONNET etc. should be defined

# --- Greeting Agent ---
greeting_agent = None
try:
    greeting_agent = Agent(
        # Using a potentially different/cheaper model for a simple task
        model=LiteLlm(model=MODEL_GPT_4O),
        name="greeting_agent",
        instruction="You are the Greeting Agent. Your ONLY task is to provide a friendly greeting to the user. "
                    "Use the 'say_hello' tool to generate the greeting. "
                    "If the user provides their name, make sure to pass it to the tool. "
                    "Do not engage in any other conversation or tasks.",
        description="Handles simple greetings and hellos using the 'say_hello' tool.", # Crucial for delegation
        tools=[say_hello],
    )
    print(f"✅ Agent '{greeting_agent.name}' created using model '{MODEL_GPT_4O}'.")
except Exception as e:
    print(f"❌ Could not create Greeting agent. Check API Key ({MODEL_GPT_4O}). Error: {e}")

# --- Farewell Agent ---
farewell_agent = None
try:
    farewell_agent = Agent(
        # Can use the same or a different model
        model=LiteLlm(model=MODEL_GPT_4O), # Sticking with GPT for this example
        name="farewell_agent",
        instruction="You are the Farewell Agent. Your ONLY task is to provide a polite goodbye message. "
                    "Use the 'say_goodbye' tool when the user indicates they are leaving or ending the conversation "
                    "(e.g., using words like 'bye', 'goodbye', 'thanks bye', 'see you'). "
                    "Do not perform any other actions.",
        description="Handles simple farewells and goodbyes using the 'say_goodbye' tool.", # Crucial for delegation
        tools=[say_goodbye],
    )
    print(f"✅ Agent '{farewell_agent.name}' created using model '{MODEL_GPT_4O}'.")
except Exception as e:
    print(f"❌ Could not create Farewell agent. Check API Key ({MODEL_GPT_4O}). Error: {e}")

```

---

### **3. 定义带子智能体的根智能体**

现在，我们升级`weather_agent`。关键变化是：

* 添加`sub_agents`参数：我们传递包含刚创建的`greeting_agent`和`farewell_agent`实例的列表  
* 更新`instruction`：我们明确告诉根智能体*关于*其子智能体以及*何时*应将任务委派给它们  

**关键概念：自动委派（Auto Flow）** 通过提供`sub_agents`列表，ADK启用自动委派。当根智能体收到用户查询时，其大模型不仅考虑自己的指令和工具，还考虑每个子智能体的`description`。如果大模型确定查询更符合子智能体的描述能力（如"处理简单问候"），它将自动生成特殊内部动作以*转移控制*给该子智能体处理该轮次。子智能体然后使用自己的模型、指令和工具处理查询。

**最佳实践**：确保根智能体的指令明确指导其委派决策。按名称提及子智能体，并描述应发生委派的条件。

```py
# @title Define the Root Agent with Sub-Agents

# Ensure sub-agents were created successfully before defining the root agent.
# Also ensure the original 'get_weather' tool is defined.
root_agent = None
runner_root = None # Initialize runner

if greeting_agent and farewell_agent and 'get_weather' in globals():
    # Let's use a capable Gemini model for the root agent to handle orchestration
    root_agent_model = MODEL_GEMINI_2_0_FLASH

    weather_agent_team = Agent(
        name="weather_agent_v2", # Give it a new version name
        model=root_agent_model,
        description="The main coordinator agent. Handles weather requests and delegates greetings/farewells to specialists.",
        instruction="You are the main Weather Agent coordinating a team. Your primary responsibility is to provide weather information. "
                    "Use the 'get_weather' tool ONLY for specific weather requests (e.g., 'weather in London'). "
                    "You have specialized sub-agents: "
                    "1. 'greeting_agent': Handles simple greetings like 'Hi', 'Hello'. Delegate to it for these. "
                    "2. 'farewell_agent': Handles simple farewells like 'Bye', 'See you'. Delegate to it for these. "
                    "Analyze the user's query. If it's a greeting, delegate to 'greeting_agent'. If it's a farewell, delegate to 'farewell_agent'. "
                    "If it's a weather request, handle it yourself using 'get_weather'. "
                    "For anything else, respond appropriately or state you cannot handle it.",
        tools=[get_weather], # Root agent still needs the weather tool for its core task
        # Key change: Link the sub-agents here!
        sub_agents=[greeting_agent, farewell_agent]
    )
    print(f"✅ Root Agent '{weather_agent_team.name}' created using model '{root_agent_model}' with sub-agents: {[sa.name for sa in weather_agent_team.sub_agents]}")

else:
    print("❌ Cannot create root agent because one or more sub-agents failed to initialize or 'get_weather' tool is missing.")
    if not greeting_agent: print(" - Greeting Agent is missing.")
    if not farewell_agent: print(" - Farewell Agent is missing.")
    if 'get_weather' not in globals(): print(" - get_weather function is missing.")



```

---

### **4. 与智能体团队交互**

现在我们已经定义了根智能体(`weather_agent_team` - *注意：确保此变量名与之前代码块中定义的匹配，可能是`# @title Define the Root Agent with Sub-Agents`，可能命名为`root_agent`*)及其专用子智能体，让我们测试委派机制。

以下代码块将：

1. 定义`async`函数`run_team_conversation`  
2. 在此函数内，为此次测试运行创建*新的专用*`InMemorySessionService`和特定会话(`session_001_agent_team`)。这隔离了测试团队动态的对话历史  
3. 创建`Runner`(`runner_agent_team`)，配置为使用我们的`weather_agent_team`（根智能体）和专用会话服务  
4. 使用更新后的`call_agent_async`函数向`runner_agent_team`发送不同类型的查询（问候、天气请求、告别）。我们明确传递此特定测试的runner、用户ID和会话ID  
5. 立即执行`run_team_conversation`函数  

我们预期以下流程：

1. "Hello there!"查询交给`runner_agent_team`  
2. 根智能体(`weather_agent_team`)接收它，基于其指令和`greeting_agent`的描述委派任务  
3. `greeting_agent`处理查询，调用其`say_hello`工具并生成响应  
4. "What is the weather in New York?"查询*不*委派，由根智能体直接使用其`get_weather`工具处理  
5. "Thanks, bye!"查询委派给`farewell_agent`，它使用其`say_goodbye`工具  

```py
# @title Interact with the Agent Team

# Ensure the root agent (e.g., 'weather_agent_team' or 'root_agent' from the previous cell) is defined.
# Ensure the call_agent_async function is defined.

# Check if the root agent variable exists before defining the conversation function
root_agent_var_name = 'root_agent' # Default name from Step 3 guide
if 'weather_agent_team' in globals(): # Check if user used this name instead
    root_agent_var_name = 'weather_agent_team'
elif 'root_agent' not in globals():
    print("⚠️ Root agent ('root_agent' or 'weather_agent_team') not found. Cannot define run_team_conversation.")
    # Assign a dummy value to prevent NameError later if the code block runs anyway
    root_agent = None

if root_agent_var_name in globals() and globals()[root_agent_var_name]:
    async def run_team_conversation():
        print("\n--- Testing Agent Team Delegation ---")
        # InMemorySessionService is simple, non-persistent storage for this tutorial.
        session_service = InMemorySessionService()

        # Define constants for identifying the interaction context
        APP_NAME = "weather_tutorial_agent_team"
        USER_ID = "user_1_agent_team"
        SESSION_ID = "session_001_agent_team" # Using a fixed ID for simplicity

        # Create the specific session where the conversation will happen
        session = session_service.create_session(
            app_name=APP_NAME,
            user_id=USER_ID,
            session_id=SESSION_ID
        )
        print(f"Session created: App='{APP_NAME}', User='{USER_ID}', Session='{SESSION_ID}'")

        # --- Get the actual root agent object ---
        # Use the determined variable name
        actual_root_agent = globals()[root_agent_var_name]

        # Create a runner specific to this agent team test
        runner_agent_team = Runner(
            agent=actual_root_agent, # Use the root agent object
            app_name=APP_NAME,       # Use the specific app name
            session_service=session_service # Use the specific session service
            )
        # Corrected print statement to show the actual root agent's name
        print(f"Runner created for agent '{actual_root_agent.name}'.")

        # Always interact via the root agent's runner, passing the correct IDs
        await call_agent_async(query = "Hello there!",
                               runner=runner_agent_team,
                               user_id=USER_ID,
                               session_id=SESSION_ID)
        await call_agent_async(query = "What is the weather in New York?",
                               runner=runner_agent_team,
                               user_id=USER_ID,
                               session_id=SESSION_ID)
        await call_agent_async(query = "Thanks, bye!",
                               runner=runner_agent_team,
                               user_id=USER_ID,
                               session_id=SESSION_ID)

    # Execute the conversation
    # Note: This may require API keys for the models used by root and sub-agents!
    await run_team_conversation()
else:
    print("\n⚠️ Skipping agent team conversation as the root agent was not successfully defined in the previous step.")

```

---

仔细观察输出日志，特别是`--- Tool: ... called ---`消息。你应该观察到：

* 对于"Hello there!"，调用了`say_hello`工具（表明`greeting_agent`处理了它）  
* 对于"What is the weather in New York?"，调用了`get_weather`工具（表明根智能体处理了它）  
* 对于"Thanks, bye!"，调用了`say_goodbye`工具（表明`farewell_agent`处理了它）  

这确认了成功的**自动委派**！根智能体在其指令和`sub_agents`的`description`指导下，正确将用户请求路由到团队内适当的专家智能体。

你现在已用多个协作智能体构建了你的应用。这种模块化设计是构建更复杂、更有能力的智能体系统的基础。下一步，我们将使用会话状态赋予智能体跨轮次记忆信息的能力。

## 第4步：通过会话状态添加记忆与个性化

到目前为止，我们的智能体团队可以通过委派处理不同任务，但每次交互都是全新的——智能体没有记忆过去对话或会话中的用户偏好。要创建更复杂和上下文感知的体验，智能体需要**记忆**。ADK通过**会话状态**提供这一点。

**什么是会话状态？**

* 它是绑定到特定用户会话（由`APP_NAME`、`USER_ID`、`SESSION_ID`标识）的Python字典(`session.state`)  
* 它在会话内*跨多个对话轮次*持久保存信息  
* 智能体和工具可以读取和写入此状态，使它们能记住细节、调整行为并个性化响应  

**智能体如何与状态交互：**

1. **`ToolContext`（主要方法）**：工具可以接受`ToolContext`对象（如果声明为最后一个参数，ADK会自动提供）。此对象通过`tool_context.state`提供对会话状态的直接访问，允许工具在执行*期间*读取偏好或保存结果  
2. **`output_key`（自动保存智能体响应）**：可以配置`Agent`带有`output_key="your_key"`。ADK然后将自动保存智能体每轮次的最终文本响应到`session.state["your_key"]`  

**在本步骤中，我们将增强天气机器人团队：**

1. 使用**新**`InMemorySessionService`隔离演示状态  
2. 用用户对`temperature_unit`的偏好初始化会话状态  
3. 创建天气工具的状态感知版本(`get_weather_stateful`)，通过`ToolContext`读取此偏好并调整其输出格式（摄氏度/华氏度）  
4. 更新根智能体使用此状态感知工具，并配置`output_key`自动将其最终天气报告保存到会话状态  
5. 运行对话观察初始状态如何影响工具，手动状态更改如何改变后续行为，以及`output_key`如何持久保存智能体响应  

---

### **1. 初始化新会话服务和状态**

为了清晰演示状态管理而不受之前步骤干扰，我们将实例化新的`InMemorySessionService`。我们还将创建一个会话，其初始状态定义用户的温度单位偏好。

```py
# @title 1. Initialize New Session Service and State

# Import necessary session components
from google.adk.sessions import InMemorySessionService

# Create a NEW session service instance for this state demonstration
session_service_stateful = InMemorySessionService()
print("✅ New InMemorySessionService created for state demonstration.")

# Define a NEW session ID for this part of the tutorial
SESSION_ID_STATEFUL = "session_state_demo_001"
USER_ID_STATEFUL = "user_state_demo"

# Define initial state data - user prefers Celsius initially
initial_state = {
    "user_preference_temperature_unit": "Celsius"
}

# Create the session, providing the initial state
session_stateful = session_service_stateful.create_session(
    app_name=APP_NAME, # Use the consistent app name
    user_id=USER_ID_STATEFUL,
    session_id=SESSION_ID_STATEFUL,
    state=initial_state # <<< Initialize state during creation
)
print(f"✅ Session '{SESSION_ID_STATEFUL}' created for user '{USER_ID_STATEFUL}'.")

# Verify the initial state was set correctly
retrieved_session = session_service_stateful.get_session(app_name=APP_NAME,
                                                         user_id=USER_ID_STATEFUL,
                                                         session_id = SESSION_ID_STATEFUL)
print("\n--- Initial Session State ---")
if retrieved_session:
    print(retrieved_session.state)
else:
    print("Error: Could not retrieve session.")

```

---

### **2. 创建状态感知天气工具**

现在，我们创建天气工具的新版本。其关键特性是接受`tool_context: ToolContext`，这允许它访问`tool_context.state`。它将读取`user_preference_temperature_unit`并相应格式化温度。

**关键概念：`ToolContext`** 此对象是你的工具逻辑与会话上下文交互的桥梁，包括读取和写入状态变量。如果定义为工具函数的最后一个参数，ADK会自动注入它。

**最佳实践**：从状态读取时，使用`dictionary.get('key', default_value)`处理键可能尚不存在的情况，确保你的工具不会崩溃。

```py
# @title 2. Create State-Aware Weather Tool
from google.adk.tools.tool_context import ToolContext

def get_weather_stateful(city: str, tool_context: ToolContext) -> dict:
    """Retrieves weather, converts temp unit based on session state."""
    print(f"--- Tool: get_weather_stateful called for {city} ---")

    # --- Read preference from state ---
    preferred_unit = tool_context.state.get("user_preference_temperature_unit", "Celsius") # Default to Celsius
    print(f"--- Tool: Reading state 'user_preference_temperature_unit': {preferred_unit} ---")

    city_normalized = city.lower().replace(" ", "")

    # Mock weather data (always stored in Celsius internally)
    mock_weather_db = {
        "newyork": {"temp_c": 25, "condition": "sunny"},
        "london": {"temp_c": 15, "condition": "cloudy"},
        "tokyo": {"temp_c": 18, "condition": "light rain"},
    }

    if city_normalized in mock_weather_db:
        data = mock_weather_db[city_normalized]
        temp_c = data["temp_c"]
        condition = data["condition"]

        # Format temperature based on state preference
        if preferred_unit == "Fahrenheit":
            temp_value = (temp_c * 9/5) + 32 # Calculate Fahrenheit
            temp_unit = "°F"
        else: # Default to Celsius
            temp_value = temp_c
            temp_unit = "°C"

        report = f"The weather in {city.capitalize()} is {condition} with a temperature of {temp_value:.0f}{temp_unit}."
        result = {"status": "success", "report": report}
        print(f"--- Tool: Generated report in {preferred_unit}. Result: {result} ---")

        # Example of writing back to state (optional for this tool)
        tool_context.state["last_city_checked_stateful"] = city
        print(f"--- Tool: Updated state 'last_city_checked_stateful': {city} ---")

        return result
    else:
        # Handle city not found
        error_msg = f"Sorry, I don't have weather information for '{city}'."
        print(f"--- Tool: City '{city}' not found. ---")
        return {"status": "error", "error_message": error_msg}

print("✅ State-aware 'get_weather_stateful' tool defined.")

```

---

### **3. 重新定义子智能体并更新根智能体**

为确保此步骤自包含并正确构建，我们首先完全按照第3步中的定义重新定义`greeting_agent`和`farewell_agent`。然后定义我们的新根智能体(`weather_agent_v4_stateful`)：

* 它使用新的`get_weather_stateful`工具  
* 它包含问候和告别子智能体用于委派  
* **关键**是设置`output_key="last_weather_report"`自动将其最终天气响应保存到会话状态  

```py
# @title 3. Redefine Sub-Agents and Update Root Agent with output_key

# Ensure necessary imports: Agent, LiteLlm, Runner
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm
from google.adk.runners import Runner
# Ensure tools 'say_hello', 'say_goodbye' are defined (from Step 3)
# Ensure model constants MODEL_GPT_4O, MODEL_GEMINI_2_5_PRO etc. are defined

# --- Redefine Greeting Agent (from Step 3) ---
greeting_agent = None
try:
    greeting_agent = Agent(
        model=MODEL_GEMINI_2_0_FLASH,
        name="greeting_agent",
        instruction="You are the Greeting Agent. Your ONLY task is to provide a friendly greeting using the 'say_hello' tool. Do nothing else.",
        description="Handles simple greetings and hellos using the 'say_hello' tool.",
        tools=[say_hello],
    )
    print(f"✅ Agent '{greeting_agent.name}' redefined.")
except Exception as e:
    print(f"❌ Could not redefine Greeting agent. Error: {e}")

# --- Redefine Farewell Agent (from Step 3) ---
farewell_agent = None
try:
    farewell_agent = Agent(
        model=MODEL_GEMINI_2_0_FLASH,
        name="farewell_agent",
        instruction="You are the Farewell Agent. Your ONLY task is to provide a polite goodbye message using the 'say_goodbye' tool. Do not perform any other actions.",
        description="Handles simple farewells and goodbyes using the 'say_goodbye' tool.",
        tools=[say_goodbye],
    )
    print(f"✅ Agent '{farewell_agent.name}' redefined.")
except Exception as e:
    print(f"❌ Could not redefine Farewell agent. Error: {e}")

# --- Define the Updated Root Agent ---
root_agent_stateful = None
runner_root_stateful = None # Initialize runner

# Check prerequisites before creating the root agent
if greeting_agent and farewell_agent and 'get_weather_stateful' in globals():

    root_agent_model = MODEL_GEMINI_2_0_FLASH # Choose orchestration model

    root_agent_stateful = Agent(
        name="weather_agent_v4_stateful", # New version name
        model=root_agent_model,
        description="Main agent: Provides weather (state-aware unit), delegates greetings/farewells, saves report to state.",
        instruction="You are the main Weather Agent. Your job is to provide weather using 'get_weather_stateful'. "
                    "The tool will format the temperature based on user preference stored in state. "
                    "Delegate simple greetings to 'greeting_agent' and farewells to 'farewell_agent'. "
                    "Handle only weather requests, greetings, and farewells.",
        tools=[get_weather_stateful], # Use the state-aware tool
        sub_agents=[greeting_agent, farewell_agent], # Include sub-agents
        output_key="last_weather_report" # <<< Auto-save agent's final weather response
    )
    print(f"✅ Root Agent '{root_agent_stateful.name}' created using stateful tool and output_key.")

    # --- Create Runner for this Root Agent & NEW Session Service ---
    runner_root_stateful = Runner(
        agent=root_agent_stateful,
        app_name=APP_NAME,
        session_service=session_service_stateful # Use the NEW stateful session service
    )
    print(f"✅ Runner created for stateful root agent '{runner_root_stateful.agent.name}' using stateful session service.")

else:
    print("❌ Cannot create stateful root agent. Prerequisites missing.")
    if not greeting_agent: print(" - greeting_agent definition missing.")
    if not farewell_agent: print(" - farewell_agent definition missing.")
    if 'get_weather_stateful' not in globals(): print(" - get_weather_stateful tool missing.")

```

---

### **4. 交互并测试状态流**

现在，让我们执行一个对话，设计用于使用`runner_root_stateful`（关联我们的状态感知智能体和`session_service_stateful`）测试状态交互。我们将使用之前定义的`call_agent_async`函数，确保传递正确的runner、用户ID(`USER_ID_STATEFUL`)和会话ID(`SESSION_ID_STATEFUL`)。

对话流程将是：

1. **检查天气（伦敦）**：`get_weather_stateful`工具应从第1节初始化的会话状态读取初始"摄氏度"偏好。根智能体的最终响应（摄氏度的天气报告）应通过`output_key`配置保存到`state['last_weather_report']`  
2. **手动更新状态**：我们将*直接修改*存储在`InMemorySessionService`实例(`session_service_stateful`)中的状态  
    * **为什么直接修改？** `session_service.get_session()`方法返回会话的*副本*。修改该副本不会影响后续智能体运行中使用的状态。对于使用`InMemorySessionService`的测试场景，我们访问内部`sessions`字典来更改`user_preference_temperature_unit`的实际存储状态值为"华氏度"。*注意：在实际应用中，状态更改通常由工具或智能体逻辑返回`EventActions(state_delta=...)`触发，而非直接手动更新*  
3. **再次检查天气（纽约）**：`get_weather_stateful`工具现在应从状态读取更新的"华氏度"偏好并相应转换温度。根智能体的*新*响应（华氏度的天气）将由于`output_key`覆盖`state['last_weather_report']`中的先前值  
4. **问候智能体**：验证委派给`greeting_agent`在状态感知操作旁边仍正常工作。此交互将成为`output_key`在此特定序列中保存的*最后*响应  
5. **检查最终状态**：对话后，我们最后一次检索会话（获取副本）并打印其状态，确认`user_preference_temperature_unit`确实是"华氏度"，观察`output_key`保存的最终值（在此运行中将是问候），并查看工具写入的`last_city_checked_stateful`值  

```py
# Ensure the stateful runner (runner_root_stateful) is available from the previous cell
# Ensure call_agent_async, USER_ID_STATEFUL, SESSION_ID_STATEFUL, APP_NAME are defined

if 'runner_root_stateful' in globals() and runner_root_stateful:
  async def run_stateful_conversation():
      print("\n--- Testing State: Temp Unit Conversion & output_key ---")

      # 1. Check weather (Uses initial state: Celsius)
      print("--- Turn 1: Requesting weather in London (expect Celsius) ---")
      await call_agent_async(query= "What's the weather in London?",
                             runner=runner_root_stateful,
                             user_id=USER_ID_STATEFUL,
                             session_id=SESSION_ID_STATEFUL
                            )

      # 2. Manually update state preference to Fahrenheit - DIRECTLY MODIFY STORAGE
      print("\n--- Manually Updating State: Setting unit to Fahrenheit ---")
      try:
          # Access the internal storage directly - THIS IS SPECIFIC TO InMemorySessionService for testing
          stored_session = session_service_stateful.sessions[APP_NAME][USER_ID_STATEFUL][SESSION_ID_STATEFUL]
          stored_session.state["user_preference_temperature_unit"] = "Fahrenheit"
          # Optional: You might want to update the timestamp as well if any logic depends on it
          # import time
          # stored_session.last_update_time = time.time()
          print(f"--- Stored session state updated. Current 'user_preference_temperature_unit': {stored_session.state['user_preference_temperature_unit']} ---")
      except KeyError:
          print(f"--- Error: Could not retrieve session '{SESSION_ID_STATEFUL}' from internal storage for user '{USER_ID_STATEFUL}' in app '{APP_NAME}' to update state. Check IDs and if session was created. ---")
      except Exception as e:
           print(f"--- Error updating internal session state: {e} ---")

      # 3. Check weather again (Tool should now use Fahrenheit)
      # This will also update 'last_weather_report' via output_key
      print("\n--- Turn 2: Requesting weather in New York (expect Fahrenheit) ---")
      await call_agent_async(query= "Tell me the weather in New York.",
                             runner=runner_root_stateful,
                             user_id=USER_ID_STATEFUL,
                             session_id=SESSION_ID_STATEFUL
                            )

      # 4. Test basic delegation (should still work)
      # This will update 'last_weather_report' again, overwriting the NY weather report
      print("\n--- Turn 3: Sending a greeting ---")
      await call_agent_async(query= "Hi!",
                             runner=runner_root_stateful,
                             user_id=USER_ID_STATEFUL,
                             session_id=SESSION_ID_STATEFUL
                            )

  # Execute the conversation
  await run_stateful_conversation()

  # Inspect final session state after the conversation
  print("\n--- Inspecting Final Session State ---")
  final_session = session_service_stateful.get_session(app_name=APP_NAME,
                                                       user_id= USER_ID_STATEFUL,
                                                       session_id=SESSION_ID_STATEFUL)
  if final_session:
      print(f"Final Preference: {final_session.state.get('user_preference_temperature_unit')}")
      print(f"Final Last Weather Report (from output_key): {final_session.state.get('last_weather_report')}")
      print(f"Final Last City Checked (by tool): {final_session.state.get('last_city_checked_stateful')}")
      # Print full state for detailed view
      # print(f"Full State: {final_session.state}")
  else:
      print("\n❌ Error: Could not retrieve final session state.")

else:
  print("\n⚠️ Skipping state test conversation. Stateful root agent runner ('runner_root_stateful') is not available.")

```

---

通过回顾对话流程和最终会话状态打印，你可以确认：

* **状态读取**：天气工具(`get_weather_stateful`)正确从状态读取`user_preference_temperature_unit`，最初对伦敦使用"摄氏度"  
* **状态更新**：直接修改成功将存储偏好更改为"华氏度"  
* **状态读取（更新后）**：当询问纽约天气时，工具随后读取"华氏度"并执行转换  
* **工具状态写入**：工具通过`tool_context.state`成功将`last_city_checked_stateful`（第二次天气检查后的"纽约"）写入状态  
* **委派**：对"Hi!"的`greeting_agent`委派在状态修改后仍正确运行  
* **`output_key`**：`output_key="last_weather_report"`成功保存根智能体*每轮次*的*最终*响应（根智能体最终响应的轮次）。在此序列中，最后响应是问候（"Hello, there!"），因此它覆盖了状态键中的天气报告  
* **最终状态**：最终检查确认偏好持久保存为"华氏度"  

你现在已成功集成会话状态，使用`ToolContext`个性化智能体行为，为测试`InMemorySessionService`手动操作状态，并观察`output_key`如何提供简单机制将智能体的最后响应保存到状态。这种状态管理的基础理解是关键，因为我们将在下一步使用回调实现安全防护机制。

---

## 第5步：添加安全性 - 使用`before_model_callback`的输入防护

我们的智能体团队正变得更强大，能记住偏好并有效使用工具。然而，在现实场景中，我们经常需要安全机制来控制智能体行为，*在*潜在问题请求甚至到达核心大模型(LLM)之前。

ADK提供**回调**——允许你挂钩到智能体执行生命周期特定点的函数。`before_model_callback`特别适用于输入安全。

**什么是`before_model_callback`？**

* 它是你定义的Python函数，ADK*刚好在*智能体将其编译的请求（包括对话历史、指令和最新用户消息）发送到底层大模型之前执行  
* **目的**：检查请求，必要时修改它，或基于预定义规则完全阻止它  

**常见用例：**

* **输入验证/过滤**：检查用户输入是否符合标准或包含不允许的内容（如PII或关键词）  
* **防护**：防止有害、离题或违反策略的请求被大模型处理  
* **动态提示修改**：在发送前向大模型请求上下文添加及时信息（如来自会话状态的信息）  

**工作原理：**

1. 定义接受`callback_context: CallbackContext`和`llm_request: LlmRequest`的函数  
   * `callback_context`：提供对智能体信息、会话状态(`callback_context.state`)等的访问  
   * `llm_request`：包含打算发送给大模型的完整负载(`contents`、`config`)  
2. 在函数内部：  
   * **检查**：检查`llm_request.contents`（特别是最后用户消息）  
   * **修改（谨慎使用）**：你可以更改`llm_request`的部分内容  
   * **阻止（防护）**：返回`LlmResponse`对象。ADK将立即发送此响应*跳过*该轮次的大模型调用  
   * **允许**：返回`None`。ADK继续使用（可能修改后的）请求调用大模型  

**在本步骤中，我们将：**

1. 定义`before_model_callback`函数(`block_keyword_guardrail`)，检查用户输入中特定关键词("BLOCK")  
2. 更新我们的状态感知根智能体(`weather_agent_v4_stateful`来自第4步)使用此回调  
3. 创建与此更新智能体关联的新runner，但使用*相同的状态感知会话服务*以保持状态连续性  
4. 通过发送正常和含关键词的请求测试防护  

---

### **1. 定义防护回调函数**

此函数将检查`llm_request`内容中的最后用户消息。如果找到"BLOCK"（不区分大小写），它构建并返回`LlmResponse`以阻止流程；否则返回`None`。

```py
# @title 1. Define the before_model_callback Guardrail

# Ensure necessary imports are available
from google.adk.agents.callback_context import CallbackContext
from google.adk.models.llm_request import LlmRequest
from google.adk.models.llm_response import LlmResponse
from google.genai import types # For creating response content
from typing import Optional

def block_keyword_guardrail(
    callback_context: CallbackContext, llm_request: LlmRequest
) -> Optional[LlmResponse]:
    """
    Inspects the latest user message for 'BLOCK'. If found, blocks the LLM call
    and returns a predefined LlmResponse. Otherwise, returns None to proceed.
    """
    agent_name = callback_context.agent_name # Get the name of the agent whose model call is being intercepted
    print(f"--- Callback: block_keyword_guardrail running for agent: {agent_name} ---")

    # Extract the text from the latest user message in the request history
    last_user_message_text = ""
    if llm_request.contents:
        # Find the most recent message with role 'user'
        for content in reversed(llm_request.contents):
            if content.role == 'user' and content.parts:
                # Assuming text is in the first part for simplicity
                if content.parts[0].text:
                    last_user_message_text = content.parts[0].text
                    break # Found the last user message text

    print(f"--- Callback: Inspecting last user message: '{last_user_message_text[:100]}...' ---") # Log first 100 chars

    # --- Guardrail Logic ---
    keyword_to_block = "BLOCK"
    if keyword_to_block in last_user_message_text.upper(): # Case-insensitive check
        print(f"--- Callback: Found '{keyword_to_block}'. Blocking LLM call! ---")
        # Optionally, set a flag in state to record the block event
        callback_context.state["guardrail_block_keyword_triggered"] = True
        print(f"--- Callback: Set state 'guardrail_block_keyword_triggered': True ---")

        # Construct and return an LlmResponse to stop the flow and send this back instead
        return LlmResponse(
            content=types.Content(
                role="model", # Mimic a response from the agent's perspective
                parts=[types.Part(text=f"I cannot process this request because it contains the blocked keyword '{keyword_to_block}'.")],
            )
            # Note: You could also set an error_message field here if needed
        )
    else:
        # Keyword not found, allow the request to proceed to the LLM
        print(f"--- Callback: Keyword not found. Allowing LLM call for {agent_name}. ---")
        return None # Returning None signals ADK to continue normally

print("✅ block_keyword_guardrail function defined.")

```

---

### **2. 更新根智能体使用回调**

我们重新定义根智能体，添加`before_model_callback`参数并指向我们的新防护函数。为清晰起见，我们给它一个新版本名称。

*重要*：我们需要在此上下文中重新定义子智能体(`greeting_agent`、`farewell_agent`)和状态感知工具(`get_weather_stateful`)（如果它们尚未从之前步骤中获得），确保根智能体定义能访问其所有组件。

```py
# @title 2. Update Root Agent with before_model_callback


# --- Redefine Sub-Agents (Ensures they exist in this context) ---
greeting_agent = None
try:
    # Use a defined model constant
    greeting_agent = Agent(
        model=MODEL_GEMINI_2_0_FLASH,
        name="greeting_agent", # Keep original name for consistency
        instruction="You are the Greeting Agent. Your ONLY task is to provide a friendly greeting using the 'say_hello' tool. Do nothing else.",
        description="Handles simple greetings and hellos using the 'say_hello' tool.",
        tools=[say_hello],
    )
    print(f"✅ Sub-Agent '{greeting_agent.name}' redefined.")
except Exception as e:
    print(f"❌ Could not redefine Greeting agent. Check Model/API Key ({MODEL_GPT_4O}). Error: {e}")

farewell_agent = None
try:
    # Use a defined model constant
    farewell_agent = Agent(
        model=MODEL_GEMINI_2_0_FLASH,
        name="farewell_agent", # Keep original name
        instruction="You are the Farewell Agent. Your ONLY task is to provide a polite goodbye message using the 'say_goodbye' tool. Do not perform any other actions.",
        description="Handles simple farewells and goodbyes using the 'say_goodbye' tool.",
        tools=[say_goodbye],
    )
    print(f"✅ Sub-Agent '{farewell_agent.name}' redefined.")
except Exception as e:
    print(f"❌ Could not redefine Farewell agent. Check Model/API Key ({MODEL_GPT_4O}). Error: {e}")


# --- Define the Root Agent with the Callback ---
root_agent_model_guardrail = None
runner_root_model_guardrail = None

# Check all components before proceeding
if greeting_agent and farewell_agent and 'get_weather_stateful' in globals() and 'block_keyword_guardrail' in globals():

    # Use a defined model constant like MODEL_GEMINI_2_5_PRO
    root_agent_model = MODEL_GEMINI_2_0_FLASH

    root_agent_model_guardrail = Agent(
        name="weather_agent_v5_model_guardrail", # New version name for clarity
        model=root_agent_model,
        description="Main agent: Handles weather, delegates greetings/farewells, includes input keyword guardrail.",
        instruction="You are the main Weather Agent. Provide weather using 'get_weather_stateful'. "
                    "Delegate simple greetings to 'greeting_agent' and farewells to 'farewell_agent'. "
                    "Handle only weather requests, greetings, and farewells.",
        tools=[get_weather],
        sub_agents=[greeting_agent, farewell_agent], # Reference the redefined sub-agents
        output_key="last_weather_report", # Keep output_key from Step 4
        before_model_callback=block_keyword_guardrail # <<< Assign the guardrail callback
    )
    print(f"✅ Root Agent '{root_agent_model_guardrail.name}' created with before_model_callback.")

    # --- Create Runner for this Agent, Using SAME Stateful Session Service ---
    # Ensure session_service_stateful exists from Step 4
    if 'session_service_stateful' in globals():
        runner_root_model_guardrail = Runner(
            agent=root_agent_model_guardrail,
            app_name=APP_NAME, # Use consistent APP_NAME
            session_service=session_service_stateful # <<< Use the service from Step 4
        )
        print(f"✅ Runner created for guardrail agent '{runner_root_model_guardrail.agent.name}', using stateful session service.")
    else:
        print("❌ Cannot create runner. 'session_service_stateful' from Step 4 is missing.")

else:
    print("❌ Cannot create root agent with model guardrail. One or more prerequisites are missing or failed initialization:")
    if not greeting_agent: print("   - Greeting Agent")
    if not farewell_agent: print("   - Farewell Agent")
    if 'get_weather_stateful' not in globals(): print("   - 'get_weather_stateful' tool")
    if 'block_keyword_guardrail' not in globals(): print("   - 'block_keyword_guardrail' callback")

```

---

### **3. 交互测试防护**

让我们测试防护行为。我们将使用与第4步*相同的会话*(`SESSION_ID_STATEFUL`)，以显示状态在这些更改中持续存在。

1. 发送正常天气请求（应通过防护并执行）  
2. 发送包含"BLOCK"的请求（应被回调拦截）  
3. 发送问候（应通过根智能体的防护，被委派并正常执行）  

```py
# @title 3. Interact to Test the Model Input Guardrail

# Ensure the runner for the guardrail agent is available
if runner_root_model_guardrail:
  async def run_guardrail_test_conversation():
      print("\n--- Testing Model Input Guardrail ---")

      # Use the runner for the agent with the callback and the existing stateful session ID
      interaction_func = lambda query: call_agent_async(query,
      runner_root_model_guardrail, USER_ID_STATEFUL, SESSION_ID_STATEFUL # <-- Pass correct IDs
  )
      # 1. Normal request (Callback allows, should use Fahrenheit from Step 4 state change)
      await interaction_func("What is the weather in London?")

      # 2. Request containing the blocked keyword
      await interaction_func("BLOCK the request for weather in Tokyo")

      # 3. Normal greeting (Callback allows root agent, delegation happens)
      await interaction_func("Hello again")


  # Execute the conversation
  await run_guardrail_test_conversation()

  # Optional: Check state for the trigger flag set by the callback
  final_session = session_service_stateful.get_session(app_name=APP_NAME,
                                                       user_id=USER_ID_STATEFUL,
                                                       session_id=SESSION_ID_STATEFUL)
  if final_session:
      print("\n--- Final Session State (After Guardrail Test) ---")
      print(f"Guardrail Triggered Flag: {final_session.state.get('guardrail_block_keyword_triggered')}")
      print(f"Last Weather Report: {final_session.state.get('last_weather_report')}") # Should be London weather
      print(f"Temperature Unit: {final_session.state.get('user_preference_temperature_unit')}") # Should be Fahrenheit
  else:
      print("\n❌ Error: Could not retrieve final session state.")

else:
  print("\n⚠️ Skipping model guardrail test. Runner ('runner_root_model_guardrail') is not available.")


```

---

观察执行流程：

1. **伦敦天气**：回调为`weather_agent_v5_model_guardrail`运行，检查消息，打印"未找到关键词。允许LLM调用。"并返回`None`。智能体继续，调用`get_weather_stateful`工具（使用第4步状态更改的"华氏度"偏好）并返回天气。此响应通过`output_key`更新`last_weather_report`  
2. **BLOCK请求**：回调再次为`weather_agent_v5_model_guardrail`运行，检查消息，找到"BLOCK"，打印"阻止LLM调用！"，设置状态标志，并返回预定义的`LlmResponse`。智能体的底层大模型*从未*为此轮次调用。用户看到回调的阻止消息  
3. **再次问候**：回调为`weather_agent_v5_model_guardrail`运行，允许请求。根智能体然后委派给`greeting_agent`。*注意：定义在根智能体上的`before_model_callback`不会自动应用于子智能体*。`greeting_agent`正常进行，调用其`say_hello`工具并返回问候  

你已成功实现输入安全层！`before_model_callback`提供强大机制在昂贵或潜在风险的大模型调用前执行规则和控制智能体行为。接下来，我们将应用类似概念为工具使用本身添加防护