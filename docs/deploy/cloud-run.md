# 部署至 Cloud Run

[Cloud Run](https://cloud.google.com/run) 是 Google 提供的全托管平台，可直接在谷歌可扩展的基础架构上运行您的代码。

您可以通过两种方式部署智能体：使用 `adk deploy cloud_run` 命令（推荐），或通过 Cloud Run 使用 `gcloud run deploy` 命令。

## 智能体示例

以下命令示例均基于 [LLM 智能体](../agents/llm-agents.md) 页面定义的 `capital_agent` 示例。假设该示例位于 `capital_agent` 目录中。

部署前请确认您的智能体代码配置如下：

1. 智能体代码位于代理目录中名为 `agent.py` 的文件
2. 智能体变量命名为 `root_agent`
3. `__init__.py` 文件位于智能体目录中，且包含 `from . import agent`

## 环境变量

请按照 [设置与安装](../get-started/installation.md) 指南配置环境变量。

```bash
export GOOGLE_CLOUD_PROJECT=your-project-id
export GOOGLE_CLOUD_LOCATION=us-central1 # Or your preferred location
export GOOGLE_GENAI_USE_VERTEXAI=True
```

*(将 `your-project-id` 替换为您的实际 GCP 项目 ID)*

## 部署命令

=== "adk CLI"

    ### adk CLI

    `adk deploy cloud_run` 命令可将智能体代码部署至 Google Cloud Run。

    部署前请确保已完成 Google Cloud 认证（`gcloud auth login` 和 `gcloud config set project <your-project-id>`）。

    #### 设置环境变量

    此步骤可选但推荐：设置环境变量可使部署命令更简洁。

    ```bash
    # 设置 Google Cloud 项目 ID
    export GOOGLE_CLOUD_PROJECT="your-gcp-project-id"

    # 设置目标 Google Cloud 位置
    export GOOGLE_CLOUD_LOCATION="us-central1" # 示例位置

    # 设置智能体代码目录路径
    export AGENT_PATH="./capital_agent" # 假设 capital_agent 位于当前目录

    # 设置 Cloud Run 服务名称（可选）
    export SERVICE_NAME="capital-agent-service"

    # 设置应用名称（可选）
    export APP_NAME="capital-agent-app"
    ```

    #### 命令用法

    ##### 最简命令

    ```bash
    adk deploy cloud_run \
    --project=$GOOGLE_CLOUD_PROJECT \
    --region=$GOOGLE_CLOUD_LOCATION \
    $AGENT_PATH
    ```

    ##### 完整命令（含可选参数）

    ```bash
    adk deploy cloud_run \
    --project=$GOOGLE_CLOUD_PROJECT \
    --region=$GOOGLE_CLOUD_LOCATION \
    --service_name=$SERVICE_NAME \
    --app_name=$APP_NAME \
    --with_ui \
    $AGENT_PATH
    ```

    ##### 参数说明

    * `AGENT_PATH`:（必填）指定智能体源代码目录路径的位置参数（如示例中的 `$AGENT_PATH` 或 `capital_agent/`）。该目录必须至少包含 `__init__.py` 和主智能体文件（如 `agent.py`）。

    ##### 选项说明

    * `--project TEXT`:（必填）Google Cloud 项目 ID（如 `$GOOGLE_CLOUD_PROJECT`）
    * `--region TEXT`:（必填）部署目标区域（如 `$GOOGLE_CLOUD_LOCATION`、`us-central1`）
    * `--service_name TEXT`:（可选）Cloud Run 服务名称（如 `$SERVICE_NAME`），默认为 `adk-default-service-name`
    * `--app_name TEXT`:（可选）ADK API 服务器应用名称（如 `$APP_NAME`），默认为 `AGENT_PATH` 指定目录的名称（如 `AGENT_PATH` 为 `./capital_agent` 时默认为 `capital_agent`）
    * `--agent_engine_id TEXT`:（可选）若通过 Vertex AI Agent Engine 使用托管会话服务，请在此提供资源 ID
    * `--port INTEGER`:（可选）ADK API 服务器在容器内监听的端口号，默认为 8000
    * `--with_ui`:（可选）添加此标志可同时部署 ADK 开发界面。默认仅部署 API 服务器
    * `--temp_folder TEXT`:（可选）指定部署过程中生成中间文件的存储目录，默认为系统临时目录中的时间戳文件夹（注：通常仅在排查问题时需要此选项）
    * `--help`: 显示帮助信息并退出

    ##### 认证访问
    部署过程中可能会提示：`Allow unauthenticated invocations to [your-service-name] (y/N)?`

    * 输入 `y` 允许无需认证即可公开访问智能体 API 端点
    * 输入 `N`（或直接回车选择默认值）则要求认证（如"测试智能体"章节所示需使用身份令牌）

    成功执行后，命令将把智能体部署至 Cloud Run 并返回服务 URL。

=== "gcloud CLI"

    ### gcloud CLI

    您也可以使用标准 `gcloud run deploy` 命令配合 `Dockerfile` 进行部署。相比 `adk` 命令，此方法需要更多手动配置，但灵活性更高，特别适合将智能体嵌入自定义 [FastAPI](https://fastapi.tiangolo.com/) 应用的场景。

    部署前请确保已完成 Google Cloud 认证（`gcloud auth login` 和 `gcloud config set project <your-project-id>`）。

    #### 项目结构

    按如下结构组织项目文件：

    ```txt
    your-project-directory/
    ├── capital_agent/
    │   ├── __init__.py
    │   └── agent.py       # 智能体代码（参见"智能体示例"标签页）
    ├── main.py            # FastAPI 应用入口
    ├── requirements.txt   # Python 依赖项
    └── Dockerfile         # 容器构建指令
    ```

    在 `your-project-directory/` 根目录创建以下文件（`main.py`、`requirements.txt`、`Dockerfile`）。

    #### 代码文件

    1. 此文件使用 ADK 的 `get_fast_api_app()` 配置 FastAPI 应用：

        ```python title="main.py"
        import os

        import uvicorn
        from fastapi import FastAPI
        from google.adk.cli.fast_api import get_fast_api_app

        # 获取 main.py 所在目录
        AGENT_DIR = os.path.dirname(os.path.abspath(__file__))
        # 会话数据库 URL 示例（如 SQLite）
        SESSION_DB_URL = "sqlite:///./sessions.db"
        # CORS 允许的源示例
        ALLOWED_ORIGINS = ["http://localhost", "http://localhost:8080", "*"]
        # 如需提供网页界面则设为 True，否则为 False
        SERVE_WEB_INTERFACE = True

        # 调用函数获取 FastAPI 应用实例
        # 确保代理目录名称（'capital_agent'）与您的代理文件夹匹配
        app: FastAPI = get_fast_api_app(
            agent_dir=AGENT_DIR,
            session_db_url=SESSION_DB_URL,
            allow_origins=ALLOWED_ORIGINS,
            web=SERVE_WEB_INTERFACE,
        )

        # 如需添加更多 FastAPI 路由或配置，可在下方添加
        # 示例：
        # @app.get("/hello")
        # async def read_root():
        #     return {"Hello": "World"}

        if __name__ == "__main__":
            # 使用 Cloud Run 提供的 PORT 环境变量，默认 8080
            uvicorn.run(app, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
        ```

        *注：我们将 `agent_dir` 指定为 `main.py` 所在目录，并使用 `os.environ.get("PORT", 8080)` 确保 Cloud Run 兼容性。*

    2. 列出必要的 Python 包：

        ```txt title="requirements.txt"
        google_adk
        # 添加智能体所需的其他依赖项
        ```

    3. 定义容器镜像：

        ```dockerfile title="Dockerfile"
        FROM python:3.13-slim
        WORKDIR /app

        COPY requirements.txt .
        RUN pip install --no-cache-dir -r requirements.txt

        RUN adduser --disabled-password --gecos "" myuser && \
            chown -R myuser:myuser /app

        COPY . .

        USER myuser

        ENV PATH="/home/myuser/.local/bin:$PATH"

        CMD ["sh", "-c", "uvicorn main:app --host 0.0.0.0 --port $PORT"]
        ```

    #### 使用 `gcloud` 部署

    在终端中导航至 `your-project-directory` 目录。

    ```bash
    gcloud run deploy capital-agent-service \
    --source . \
    --region $GOOGLE_CLOUD_LOCATION \
    --project $GOOGLE_CLOUD_PROJECT \
    --allow-unauthenticated \
    --set-env-vars="GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION,GOOGLE_GENAI_USE_VERTEXAI=$GOOGLE_GENAI_USE_VERTEXAI"
    # 添加智能体可能需要的其他环境变量
    ```

    * `capital-agent-service`: Cloud Run 服务名称
    * `--source .`: 指示 gcloud 使用当前目录的 Dockerfile 构建容器镜像
    * `--region`: 指定部署区域
    * `--project`: 指定 GCP 项目
    * `--allow-unauthenticated`: 允许公开访问服务。如需私有服务请移除此标志
    * `--set-env-vars`: 向运行容器传递必要的环境变量。请确保包含 ADK 和智能体所需的所有变量（如未使用应用默认凭据需包含 API 密钥）

    `gcloud` 将构建 Docker 镜像，推送至 Google Artifact Registry，并部署到 Cloud Run。完成后将输出已部署服务的 URL。

    完整部署选项请参阅 [`gcloud run deploy` 参考文档](https://cloud.google.com/sdk/gcloud/reference/run/deploy)。

## 测试智能体

智能体部署至 Cloud Run 后，您可通过部署的 UI（如启用）或使用 `curl` 等工具直接与其 API 端点交互。测试需要用到部署后提供的服务 URL。

=== "UI 测试"

    ### UI 测试

    若您在部署时启用了 UI：

    *   **adk CLI:** 部署时包含 `--with_ui` 标志
    *   **gcloud CLI:** 在 `main.py` 中设置 `SERVE_WEB_INTERFACE = True`

    您只需在浏览器中访问部署后提供的 Cloud Run 服务 URL 即可测试智能体。

    ```bash
    # URL 格式示例
    # https://your-service-name-abc123xyz.a.run.app
    ```

    ADK 开发界面支持直接通过浏览器与智能体交互、管理会话及查看执行详情。

    验证智能体功能是否正常：

    1. 从下拉菜单中选择您的智能体
    2. 输入消息并确认收到预期响应

    如遇异常行为，请查看 [Cloud Run](https://console.cloud.google.com/run) 控制台日志。

=== "API 测试 (curl)"

    ### API 测试 (curl)

    您可以使用 `curl` 等工具与智能体 API 端点交互。这在程序化交互或未启用 UI 的部署中特别有用。

    测试需要部署后提供的服务 URL，如果服务未设置为允许未认证访问，则可能还需要身份令牌。

    #### 设置应用 URL

    将示例 URL 替换为您实际部署的 Cloud Run 服务 URL。

    ```bash
    export APP_URL="YOUR_CLOUD_RUN_SERVICE_URL"
    # 示例：export APP_URL="https://adk-default-service-name-abc123xyz.a.run.app"
    ```

    #### 获取身份令牌（如需）

    如果服务需要认证（即未使用 `--allow-unauthenticated` 配合 `gcloud` 或在 `adk` 提示时回答'N'），请获取身份令牌。

    ```bash
    export TOKEN=$(gcloud auth print-identity-token)
    ```

    *如服务允许未认证访问，可省略以下 `curl` 命令中的 `-H "Authorization: Bearer $TOKEN"` 请求头。*

    #### 列出可用应用

    验证已部署的应用名称。

    ```bash
    curl -X GET -H "Authorization: Bearer $TOKEN" $APP_URL/list-apps
    ```

    *（后续命令中的 `app_name` 可根据此输出调整。默认通常为代理目录名称，如 `capital_agent`）*。

    #### 创建或更新会话

    初始化或更新特定用户和会话的状态。如应用名称不同请替换 `capital_agent`。`user_123` 和 `session_abc` 为示例标识符，可替换为您所需的用户和会话 ID。

    ```bash
    curl -X POST -H "Authorization: Bearer $TOKEN" \
        $APP_URL/apps/capital_agent/users/user_123/sessions/session_abc \
        -H "Content-Type: application/json" \
        -d '{"state": {"preferred_language": "English", "visit_count": 5}}'
    ```

    #### 运行智能体

    向智能体发送提示词。请替换 `capital_agent` 为您的应用名称，并按需调整用户/会话 ID 及提示内容。

    ```bash
    curl -X POST -H "Authorization: Bearer $TOKEN" \
        $APP_URL/run_sse \
        -H "Content-Type: application/json" \
        -d '{
        "app_name": "capital_agent",
        "user_id": "user_123",
        "session_id": "session_abc",
        "new_message": {
            "role": "user",
            "parts": [{
            "text": "What is the capital of Canada?"
            }]
        },
        "streaming": false
        }'
    ```

    * 如需接收服务器发送事件 (SSE)，请设置 `"streaming": true`
    * 响应将包含智能体的执行事件，包括最终答案