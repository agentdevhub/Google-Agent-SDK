# 测试你的智能体

在部署智能体之前，应当进行测试以确保其功能符合预期。开发环境中最简便的测试方式是使用 `adk api_server` 命令，该命令会启动本地 FastAPI 服务器，您可以通过 cURL 命令或发送 API 请求来测试智能体。

## 本地测试

本地测试包含启动本地 API 服务器、创建会话以及向智能体发送查询。首先请确保处于正确的工作目录：

```console
parent_folder  <-- you should be here
|- my_sample_agent
  |- __init__.py
  |- .env
  |- agent.py
```

**启动本地服务器**

接下来启动本地 FastAPI 服务器：

```shell
adk api_server
```

输出应类似如下：

```shell
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

此时服务器已在本地运行，地址为 `http://0.0.0.0:8000`。

**创建新会话**

保持 API 服务器运行状态，打开新的终端窗口或标签页，使用以下命令创建智能体会话：

```shell
curl -X POST http://0.0.0.0:8000/apps/my_sample_agent/users/u_123/sessions/s_123 \
  -H "Content-Type: application/json" \
  -d '{"state": {"key1": "value1", "key2": 42}}'
```

以下是对命令的解析：

* `http://0.0.0.0:8000/apps/my_sample_agent/users/u_123/sessions/s_123`：为智能体 `my_sample_agent`（即智能体文件夹名称）创建新会话，指定用户 ID（`u_123`）和会话 ID（`s_123`）。您可以将 `my_sample_agent` 替换为智能体文件夹名称，`u_123` 替换为特定用户 ID，`s_123` 替换为特定会话 ID。
* `{"state": {"key1": "value1", "key2": 42}}`：可选参数，用于在创建会话时自定义智能体的预设状态（字典格式）。

成功创建会话后将返回会话信息，输出应类似：

```shell
{"id":"s_123","app_name":"my_sample_agent","user_id":"u_123","state":{"state":{"key1":"value1","key2":42}},"events":[],"last_update_time":1743711430.022186}
```

!!! info

    无法使用完全相同的用户 ID 和会话 ID 创建多个会话。若尝试重复创建，可能收到如下响应：`{"detail":"Session already exists: s_123"}`。解决方法包括删除现有会话（如执行 `s_123`）或改用新的会话 ID。

**发送查询**

有两种通过 POST 请求向智能体发送查询的方式，分别通过 `/run` 或 `/run_sse` 路由：

* `POST http://0.0.0.0:8000/run`：以列表形式收集所有事件并一次性返回，适合大多数用户（若无特殊需求推荐使用此方式）
* `POST http://0.0.0.0:8000/run_sse`：以服务器推送事件（Server-Sent-Events）形式返回事件流，适合需要实时获取事件通知的场景。通过 `/run_sse` 还可设置 `streaming` 为 `true` 来启用令牌级流式传输。

**使用 `/run`**

```shell
curl -X POST http://0.0.0.0:8000/run \
-H "Content-Type: application/json" \
-d '{
"app_name": "my_sample_agent",
"user_id": "u_123",
"session_id": "s_123",
"new_message": {
    "role": "user",
    "parts": [{
    "text": "Hey whats the weather in new york today"
    }]
}
}'
```

使用 `/run` 时，所有事件输出将作为列表一次性返回，输出应类似：

```shell
[{"content":{"parts":[{"functionCall":{"id":"af-e75e946d-c02a-4aad-931e-49e4ab859838","args":{"city":"new york"},"name":"get_weather"}}],"role":"model"},"invocation_id":"e-71353f1e-aea1-4821-aa4b-46874a766853","author":"weather_time_agent","actions":{"state_delta":{},"artifact_delta":{},"requested_auth_configs":{}},"long_running_tool_ids":[],"id":"2Btee6zW","timestamp":1743712220.385936},{"content":{"parts":[{"functionResponse":{"id":"af-e75e946d-c02a-4aad-931e-49e4ab859838","name":"get_weather","response":{"status":"success","report":"The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit)."}}}],"role":"user"},"invocation_id":"e-71353f1e-aea1-4821-aa4b-46874a766853","author":"weather_time_agent","actions":{"state_delta":{},"artifact_delta":{},"requested_auth_configs":{}},"id":"PmWibL2m","timestamp":1743712221.895042},{"content":{"parts":[{"text":"OK. The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit).\n"}],"role":"model"},"invocation_id":"e-71353f1e-aea1-4821-aa4b-46874a766853","author":"weather_time_agent","actions":{"state_delta":{},"artifact_delta":{},"requested_auth_configs":{}},"id":"sYT42eVC","timestamp":1743712221.899018}]
```

**使用 `/run_sse`**

```shell
curl -X POST http://0.0.0.0:8000/run_sse \
-H "Content-Type: application/json" \
-d '{
"app_name": "my_sample_agent",
"user_id": "u_123",
"session_id": "s_123",
"new_message": {
    "role": "user",
    "parts": [{
    "text": "Hey whats the weather in new york today"
    }]
},
"streaming": false
}'
```

将 `streaming` 设为 `true` 可启用令牌级流式传输，响应将以多个数据块形式返回，输出应类似：

```shell
data: {"content":{"parts":[{"functionCall":{"id":"af-f83f8af9-f732-46b6-8cb5-7b5b73bbf13d","args":{"city":"new york"},"name":"get_weather"}}],"role":"model"},"invocation_id":"e-3f6d7765-5287-419e-9991-5fffa1a75565","author":"weather_time_agent","actions":{"state_delta":{},"artifact_delta":{},"requested_auth_configs":{}},"long_running_tool_ids":[],"id":"ptcjaZBa","timestamp":1743712255.313043}

data: {"content":{"parts":[{"functionResponse":{"id":"af-f83f8af9-f732-46b6-8cb5-7b5b73bbf13d","name":"get_weather","response":{"status":"success","report":"The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit)."}}}],"role":"user"},"invocation_id":"e-3f6d7765-5287-419e-9991-5fffa1a75565","author":"weather_time_agent","actions":{"state_delta":{},"artifact_delta":{},"requested_auth_configs":{}},"id":"5aocxjaq","timestamp":1743712257.387306}

data: {"content":{"parts":[{"text":"OK. The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit).\n"}],"role":"model"},"invocation_id":"e-3f6d7765-5287-419e-9991-5fffa1a75565","author":"weather_time_agent","actions":{"state_delta":{},"artifact_delta":{},"requested_auth_configs":{}},"id":"rAnWGSiV","timestamp":1743712257.391317}
```

!!! info

    使用 `/run_sse` 时，每个事件将在生成后立即显示。

## 集成方案

ADK 通过[回调函数](../callbacks/index.md)与第三方可观测性工具集成。这些集成能捕获智能体调用与交互的详细追踪记录，对于理解行为、调试问题和评估性能至关重要。

* [Comet Opik](https://github.com/comet-ml/opik) 是开源的大模型可观测性与评估平台，[原生支持 ADK](https://www.comet.com/docs/opik/tracing/integrations/adk)

## 部署智能体

完成本地验证后，即可开始部署智能体。以下是可选的部署方式：

* 部署至 [Agent Engine](../deploy/agent-engine.md)，这是将 ADK 智能体部署到 Google Cloud Vertex AI 托管服务的最简途径
* 部署至 [Cloud Run](../deploy/cloud-run.md)，通过 Google Cloud 的无服务器架构全面掌控智能体的扩展与管理