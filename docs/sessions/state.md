# State：会话的临时记事板

在每个 %%PH_70462ce%%（我们的对话线程）中，**`state`** 属性就像是为该特定交互设计的专属记事板。虽然 `session.events` 保存完整历史记录，但 `session.state` 是智能体存储和更新对话过程中所需动态细节的地方。

## 什么是 `session.state`？

从概念上讲，`session.state` 是一个存储键值对的字典。它专为智能体需要记住或跟踪的信息而设计，以使当前对话更有效：

* **个性化交互**：记住用户之前提到的偏好（例如 `'user_preference_theme': 'dark'`）  
* **跟踪任务进度**：记录多轮流程中的步骤（例如 `'booking_step': 'confirm_payment'`）  
* **积累信息**：构建列表或摘要（例如 `'shopping_cart_items': ['book', 'pen']`）  
* **做出明智决策**：存储影响下个响应的标志或值（例如 `'user_is_authenticated': True`）

### `State` 的关键特性

1. **结构：可序列化的键值对**  

    * 数据以 `key: value` 形式存储  
    * **键**：始终为字符串（`str`）。使用清晰名称（例如 `'departure_city'`，`'user:language_preference'`）  
    * **值**：必须可序列化。这意味着它们可以被 `SessionService` 轻松保存和加载。仅使用基本 Python 类型，如字符串、数字、布尔值，以及仅包含这些基本类型的简单列表或字典。（详见 API 文档）  
    * **⚠️ 避免复杂对象**：不要直接在 state 中存储不可序列化的 Python 对象（自定义类实例、函数、连接等）。如需存储，请使用简单标识符并在其他地方检索复杂对象。

2. **可变性：它会变化**  

    * `state` 的内容会随着对话进展而变化。

3. **持久性：取决于 `SessionService`**  

    * state 是否在应用重启后保留取决于所选服务：  
      * `InMemorySessionService`：非持久化。重启后 state 会丢失。  
      * `DatabaseSessionService` / `VertexAiSessionService`：持久化。state 会被可靠保存。

### 使用前缀组织 State：作用域很重要

state 键的前缀定义了它们的作用域和持久化行为，特别是在持久化服务中：

* **无前缀（会话状态）**  

    * **作用域**：特定于当前会话（`id`）  
    * **持久性**：仅当 `SessionService` 是持久化（`Database`，`VertexAI`）时才保留  
    * **用例**：跟踪当前任务进度（例如 `'current_booking_step'`），本次交互的临时标志（例如 `'needs_clarification'`）  
    * **示例**：`session.state['current_intent'] = 'book_flight'`

* **`user:` 前缀（用户状态）**  

    * **作用域**：绑定到 `user_id`，在该用户的所有会话中共享（同一 `app_name` 内）  
    * **持久性**：使用 `Database` 或 `VertexAI` 时持久化。（由 `InMemory` 存储，但重启后丢失）  
    * **用例**：用户偏好（例如 `'user:theme'`），个人资料详情（例如 `'user:name'`）  
    * **示例**：`session.state['user:preferred_language'] = 'fr'`

* **`app:` 前缀（应用状态）**  

    * **作用域**：绑定到 `app_name`，在该应用的所有用户和会话中共享  
    * **持久性**：使用 `Database` 或 `VertexAI` 时持久化。（由 `InMemory` 存储，但重启后丢失）  
    * **用例**：全局设置（例如 `'app:api_endpoint'`），共享模板  
    * **示例**：`session.state['app:global_discount_code'] = 'SAVE10'`

* **`temp:` 前缀（临时会话状态）**  

    * **作用域**：特定于当前会话处理轮次  
    * **持久性**：永不持久化。即使使用持久化服务也保证丢弃  
    * **用例**：仅立即需要的中间结果，明确不希望存储的数据  
    * **示例**：`session.state['temp:raw_api_response'] = {...}`

**智能体如何查看它**：您的智能体代码通过单一的 `session.state` 字典与组合状态交互。`SessionService` 根据前缀从正确的底层存储中获取/合并状态。

### 如何更新 State：推荐方法

State 应始终作为向会话历史添加 `Event` 的一部分，使用 `session_service.append_event()` 进行更新。这确保更改被跟踪、持久化正常工作，并且更新是线程安全的。

**1. 简单方法：`output_key`（适用于智能体文本响应）**

这是将智能体的最终文本响应直接保存到 state 的最简单方法。定义 `LlmAgent` 时，指定 `output_key`：

```py
from google.adk.agents import LlmAgent
from google.adk.sessions import InMemorySessionService, Session
from google.adk.runners import Runner
from google.genai.types import Content, Part

# Define agent with output_key
greeting_agent = LlmAgent(
    name="Greeter",
    model="gemini-2.0-flash", # Use a valid model
    instruction="Generate a short, friendly greeting.",
    output_key="last_greeting" # Save response to state['last_greeting']
)

# --- Setup Runner and Session ---
app_name, user_id, session_id = "state_app", "user1", "session1"
session_service = InMemorySessionService()
runner = Runner(
    agent=greeting_agent,
    app_name=app_name,
    session_service=session_service
)
session = session_service.create_session(app_name=app_name, 
                                        user_id=user_id, 
                                        session_id=session_id)
print(f"Initial state: {session.state}")

# --- Run the Agent ---
# Runner handles calling append_event, which uses the output_key
# to automatically create the state_delta.
user_message = Content(parts=[Part(text="Hello")])
for event in runner.run(user_id=user_id, 
                        session_id=session_id, 
                        new_message=user_message):
    if event.is_final_response():
      print(f"Agent responded.") # Response text is also in event.content

# --- Check Updated State ---
updated_session = session_service.get_session(app_name, user_id, session_id)
print(f"State after agent run: {updated_session.state}")
# Expected output might include: {'last_greeting': 'Hello there! How can I help you today?'}
```

在后台，`Runner` 使用 `output_key` 创建必要的 `EventActions`，带有 `state_delta` 并调用 `append_event`。

**2. 标准方法：`EventActions.state_delta`（适用于复杂更新）**

对于更复杂的场景（更新多个键、非字符串值、特定作用域如 `user:` 或 `app:`，或与智能体最终文本不直接相关的更新），您可以在 `EventActions` 中手动构建 `state_delta`。

```py
from google.adk.sessions import InMemorySessionService, Session
from google.adk.events import Event, EventActions
from google.genai.types import Part, Content
import time

# --- Setup ---
session_service = InMemorySessionService()
app_name, user_id, session_id = "state_app_manual", "user2", "session2"
session = session_service.create_session(
    app_name=app_name,
    user_id=user_id,
    session_id=session_id,
    state={"user:login_count": 0, "task_status": "idle"}
)
print(f"Initial state: {session.state}")

# --- Define State Changes ---
current_time = time.time()
state_changes = {
    "task_status": "active",              # Update session state
    "user:login_count": session.state.get("user:login_count", 0) + 1, # Update user state
    "user:last_login_ts": current_time,   # Add user state
    "temp:validation_needed": True        # Add temporary state (will be discarded)
}

# --- Create Event with Actions ---
actions_with_update = EventActions(state_delta=state_changes)
# This event might represent an internal system action, not just an agent response
system_event = Event(
    invocation_id="inv_login_update",
    author="system", # Or 'agent', 'tool' etc.
    actions=actions_with_update,
    timestamp=current_time
    # content might be None or represent the action taken
)

# --- Append the Event (This updates the state) ---
session_service.append_event(session, system_event)
print("`append_event` called with explicit state delta.")

# --- Check Updated State ---
updated_session = session_service.get_session(app_name=app_name,
                                            user_id=user_id, 
                                            session_id=session_id)
print(f"State after event: {updated_session.state}")
# Expected: {'user:login_count': 1, 'task_status': 'active', 'user:last_login_ts': <timestamp>}
# Note: 'temp:validation_needed' is NOT present.
```

**`append_event` 的作用：**

* 将 `Event` 添加到 `session.events`  
* 从事件的 `actions` 读取 `state_delta`  
* 将这些更改应用到由 `SessionService` 管理的 state，根据服务类型正确处理前缀和持久化  
* 更新会话的 `last_update_time`  
* 确保并发更新的线程安全

### ⚠️ 关于直接修改 State 的警告

避免在检索会话后直接修改 `session.state` 字典（例如 `retrieved_session.state['key'] = value`）。

**强烈不建议这样做的原因：**

1. **绕过事件历史**：更改不会记录为 `Event`，失去可审计性  
2. **破坏持久化**：以这种方式进行的更改很可能不会被 `DatabaseSessionService` 或 `VertexAiSessionService` 保存。它们依赖 `append_event` 触发保存  
3. **非线程安全**：可能导致竞态条件和更新丢失  
4. **忽略时间戳/逻辑**：不会更新 `last_update_time` 或触发相关事件逻辑

**建议**：为了可靠、可跟踪和持久化的状态管理，坚持通过 `output_key` 或 `EventActions.state_delta` 在 `append_event` 流程中更新 state。仅在读取 state 时使用直接访问。

### State 设计最佳实践回顾

* **极简主义**：仅存储必要的动态数据  
* **序列化**：使用基本的可序列化类型  
* **描述性键和前缀**：使用清晰名称和适当前缀（`user:`，`app:`，`temp:` 或无前缀）  
* **浅层结构**：尽可能避免深度嵌套  
* **标准更新流程**：依赖 `append_event`