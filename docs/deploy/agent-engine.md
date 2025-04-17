# 部署至 Vertex AI Agent Engine

[Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview) 是 Google Cloud 提供的全托管服务，可帮助开发者在生产环境中部署、管理和扩展 AI 代理。该服务负责处理生产环境的基础设施扩展，让开发者能专注于构建智能高效的应用系统。

```python
from vertexai import agent_engines

remote_app = agent_engines.create(
    agent_engine=root_agent,
    requirements=[
        "google-cloud-aiplatform[adk,agent_engines]",
    ]
)
```

## 安装 Vertex AI SDK

Agent Engine 是 Vertex AI Python SDK 的组成部分。更多信息请参阅 [Agent Engine 快速入门文档](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/quickstart)。

### 安装 Vertex AI SDK

```shell
pip install google-cloud-aiplatform[adk,agent_engines]
```

!!!info
    Agent Engine 仅支持 Python 3.9 至 3.12 版本

### 初始化配置

```py
import vertexai

PROJECT_ID = "your-project-id"
LOCATION = "us-central1"
STAGING_BUCKET = "gs://your-google-cloud-storage-bucket"

vertexai.init(
    project=PROJECT_ID,
    location=LOCATION,
    staging_bucket=STAGING_BUCKET,
)
```

关于 `LOCATION`，可查阅 [Agent Engine 支持区域列表](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview#supported-regions)。

### 创建代理

以下示例代理包含两个工具（获取天气和查询指定城市时间）：

```python
--8<-- "examples/python/snippets/get-started/multi_tool_agent/agent.py"
```

### 适配 Agent Engine 部署

使用 `reasoning_engines.AdkApp()` 封装代理以实现可部署性：

```py
from vertexai.preview import reasoning_engines

app = reasoning_engines.AdkApp(
    agent=root_agent,
    enable_tracing=True,
)
```

### 本地测试代理

部署前可先在本地验证功能。

#### 创建会话（本地）

```py
session = app.create_session(user_id="u_123")
session
```

`create_session` 的预期输出（本地）：

```console
Session(id='c6a33dae-26ef-410c-9135-b434a528291f', app_name='default-app-name', user_id='u_123', state={}, events=[], last_update_time=1743440392.8689594)
```

#### 列出会话（本地）

```py
app.list_sessions(user_id="u_123")
```

`list_sessions` 的预期输出（本地）：

```console
ListSessionsResponse(session_ids=['c6a33dae-26ef-410c-9135-b434a528291f'])
```

#### 获取特定会话（本地）

```py
session = app.get_session(user_id="u_123", session_id=session.id)
session
```

`get_session` 的预期输出（本地）：

```console
Session(id='c6a33dae-26ef-410c-9135-b434a528291f', app_name='default-app-name', user_id='u_123', state={}, events=[], last_update_time=1743681991.95696)
```

#### 向代理发送查询（本地）

```py
for event in app.stream_query(
    user_id="u_123",
    session_id=session.id,
    message="whats the weather in new york",
):
print(event)
```

`stream_query` 的预期输出（本地）：

```console
{'parts': [{'function_call': {'id': 'af-a33fedb0-29e6-4d0c-9eb3-00c402969395', 'args': {'city': 'new york'}, 'name': 'get_weather'}}], 'role': 'model'}
{'parts': [{'function_response': {'id': 'af-a33fedb0-29e6-4d0c-9eb3-00c402969395', 'name': 'get_weather', 'response': {'status': 'success', 'report': 'The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit).'}}}], 'role': 'user'}
{'parts': [{'text': 'The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit).'}], 'role': 'model'}
```

### 部署至 Agent Engine

```python
from vertexai import agent_engines

remote_app = agent_engines.create(
    agent_engine=root_agent,
    requirements=[
        "google-cloud-aiplatform[adk,agent_engines]"   
    ]
)
```

此步骤可能需要数分钟完成。

## 授予部署代理权限

在 Agent Engine 上查询代理前，需先授予托管会话使用权限。托管会话是 Agent Engine 的内置组件，用于维护对话状态。若未授予以下权限，查询时可能出现错误。

按照 [设置服务代理权限](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/set-up#service-agent) 指南，通过 [IAM 管理页面](https://console.cloud.google.com/iam-admin/iam) 授予：

*  将 Vertex AI 用户权限（`roles/aiplatform.user`）授予您的 `service-PROJECT_NUMBER@gcp-sa-aiplatform-re.iam.gserviceaccount.com` 服务账号

### 在 Agent Engine 上测试代理

#### 创建会话（远程）

```py
remote_session = remote_app.create_session(user_id="u_456")
remote_session
```

`create_session` 的预期输出（远程）：

```console
{'events': [],
'user_id': 'u_456',
'state': {},
'id': '7543472750996750336',
'app_name': '7917477678498709504',
'last_update_time': 1743683353.030133}
```

其中 `id` 为会话 ID，`app_name` 是 Agent Engine 上的代理资源 ID。

#### 列出会话（远程）

```py
remote_app.list_sessions(user_id="u_456")
```

#### 获取特定会话（远程）

```py
remote_app.get_session(user_id="u_456", session_id=remote_session["id"])
```

!!!note
    本地使用时会话 ID 存储在 `session.id`，远程使用时则存储在 `remote_session["id"]`。

#### 向代理发送查询（远程）

```py
for event in remote_app.stream_query(
    user_id="u_456",
    session_id=remote_session["id"],
    message="whats the weather in new york",
):
    print(event)
```

`stream_query` 的预期输出（远程）：

```console
{'parts': [{'function_call': {'id': 'af-f1906423-a531-4ecf-a1ef-723b05e85321', 'args': {'city': 'new york'}, 'name': 'get_weather'}}], 'role': 'model'}
{'parts': [{'function_response': {'id': 'af-f1906423-a531-4ecf-a1ef-723b05e85321', 'name': 'get_weather', 'response': {'status': 'success', 'report': 'The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit).'}}}], 'role': 'user'}
{'parts': [{'text': 'The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit).'}], 'role': 'model'}
```

## 资源清理

使用完毕后建议清理云资源。删除已部署的 Agent Engine 实例可避免产生意外费用。

```python
remote_app.delete(force=True)
```

`force=True` 将同时删除该代理衍生的所有子资源（如会话记录）。