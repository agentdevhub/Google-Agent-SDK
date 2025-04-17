# 在ADK中使用不同的大模型

Agent Development Kit (ADK) 设计具有高度灵活性，支持集成多种大模型到您的智能体中。虽然[设置基础模型](../get-started/installation.md)指南已涵盖Google Gemini模型的配置，但本文将详细介绍如何高效利用Gemini，并集成其他主流模型（包括外部托管或本地运行的模型）。

ADK主要通过两种机制实现模型集成：

1. **直接字符串/注册表方式**：适用于与Google Cloud深度集成的模型（如通过Google AI Studio或Vertex AI访问的Gemini模型）或托管在Vertex AI终端的模型。通常直接将模型名称或终端资源字符串提供给`LlmAgent`。ADK内部注册表会将该字符串解析为对应的后端客户端，通常使用`google-genai`库实现。
2. **封装类方式**：为获得更广泛的兼容性（特别是Google生态系统外的模型或需要特定客户端配置的模型，如通过LiteLLM访问的模型）。您需要实例化特定的封装类（例如`LiteLlm`），并将该对象作为`model`参数传递给您的`LlmAgent`。

以下章节将根据您的需求指导您使用这些方法。

## 使用Google Gemini模型

这是在ADK中使用Google旗舰模型最直接的方式。

**集成方法**：将模型标识字符串直接传递给`LlmAgent`（或其别名`Agent`）的`model`参数。

**后端选项与设置**：

ADK内部用于Gemini的`google-genai`库可通过Google AI Studio或Vertex AI建立连接。

!!!note "语音/视频流支持的模型"

    要在ADK中使用语音/视频流功能，需要使用支持Live API的Gemini模型。您可以在以下文档中找到支持Gemini Live API的**模型ID**：

    - [Google AI Studio: Gemini Live API](https://ai.google.dev/gemini-api/docs/models#live-api)
    - [Vertex AI: Gemini Live API](https://cloud.google.com/vertex-ai/generative-ai/docs/live-api)

### Google AI Studio

* **使用场景**：Google AI Studio是开始使用Gemini的最简单方式。只需获取[API密钥](https://aistudio.google.com/app/apikey)。最适合快速原型开发和测试。
* **设置**：通常需要将API密钥设置为环境变量：

```shell
export GOOGLE_API_KEY="YOUR_GOOGLE_API_KEY"
export GOOGLE_GENAI_USE_VERTEXAI=FALSE
```

* **模型列表**：所有可用模型请参见[Google AI开发者网站](https://ai.google.dev/gemini-api/docs/models)。

### Vertex AI

* **使用场景**：推荐用于生产环境应用，可充分利用Google云基础设施。Vertex AI上的Gemini支持企业级功能、安全性和合规控制。
* **设置**：
    * 使用应用默认凭证(ADC)进行认证：

        ```shell
        gcloud auth application-default login
        ```

    * 设置Google Cloud项目和位置：

        ```shell
        export GOOGLE_CLOUD_PROJECT="YOUR_PROJECT_ID"
        export GOOGLE_CLOUD_LOCATION="YOUR_VERTEX_AI_LOCATION" # 例如us-central1
        ```

    * 明确告知库使用Vertex AI：

        ```shell
        export GOOGLE_GENAI_USE_VERTEXAI=TRUE
        ```

* **模型列表**：可用模型ID请参见[Vertex AI文档](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/models)。

**示例**：

```python
from google.adk.agents import LlmAgent

# --- Example using a stable Gemini Flash model ---
agent_gemini_flash = LlmAgent(
    # Use the latest stable Flash model identifier
    model="gemini-2.0-flash",
    name="gemini_flash_agent",
    instruction="You are a fast and helpful Gemini assistant.",
    # ... other agent parameters
)

# --- Example using a powerful Gemini Pro model ---
# Note: Always check the official Gemini documentation for the latest model names,
# including specific preview versions if needed. Preview models might have
# different availability or quota limitations.
agent_gemini_pro = LlmAgent(
    # Use the latest generally available Pro model identifier
    model="gemini-2.5-pro-preview-03-25",
    name="gemini_pro_agent",
    instruction="You are a powerful and knowledgeable Gemini assistant.",
    # ... other agent parameters
)
```

## 通过LiteLLM使用云端及专有模型

要访问OpenAI、Anthropic（非Vertex AI）、Cohere等提供商的大量大模型，ADK通过LiteLLM库提供集成支持。

**集成方法**：实例化`LiteLlm`封装类并将其传递给`LlmAgent`的`model`参数。

**LiteLLM概述**：[LiteLLM](https://docs.litellm.ai/)作为转换层，为100+大模型提供标准化的OpenAI兼容接口。

**设置**：

1. **安装LiteLLM**：
        ```shell
        pip install litellm
        ```
2. **设置提供商API密钥**：为您计划使用的特定提供商配置环境变量。

    * *OpenAI示例*：

        ```shell
        export OPENAI_API_KEY="YOUR_OPENAI_API_KEY"
        ```

    * *Anthropic（非Vertex AI）示例*：

        ```shell
        export ANTHROPIC_API_KEY="YOUR_ANTHROPIC_API_KEY"
        ```

    * *其他提供商的环境变量名称请参考[LiteLLM提供商文档](https://docs.litellm.ai/docs/providers)*

        **示例**：

        ```python
        from google.adk.agents import LlmAgent
        from google.adk.models.lite_llm import LiteLlm

        # --- 使用OpenAI GPT-4o的智能体示例 ---
        # (需要OPENAI_API_KEY)
        agent_openai = LlmAgent(
            model=LiteLlm(model="openai/gpt-4o"), # LiteLLM模型字符串格式
            name="openai_agent",
            instruction="您是由GPT-4o驱动的助手。",
            # ... 其他智能体参数
        )

        # --- 使用Anthropic Claude Haiku（非Vertex）的智能体示例 ---
        # (需要ANTHROPIC_API_KEY)
        agent_claude_direct = LlmAgent(
            model=LiteLlm(model="anthropic/claude-3-haiku-20240307"),
            name="claude_direct_agent",
            instruction="您是由Claude Haiku驱动的助手。",
            # ... 其他智能体参数
        )
        ```

## 通过LiteLLM使用开源及本地模型

为了获得最大控制权、节省成本、保护隐私或离线使用，您可以本地运行开源模型或自行托管，并通过LiteLLM集成。

**集成方法**：实例化`LiteLlm`封装类，并配置指向您的本地模型服务器。

### Ollama集成

[Ollama](https://ollama.com/)让您可以轻松在本地运行开源模型。

**前提条件**：

1. 安装Ollama。
2. 拉取所需模型（例如Google的Gemma）：

    ```shell
    ollama pull gemma:2b
    ```

3. 确保Ollama服务器正在运行（安装后通常自动运行，或通过运行`ollama serve`启动）。

    **示例**：

    ```python
    from google.adk.agents import LlmAgent
    from google.adk.models.lite_llm import LiteLlm

    # --- 使用通过Ollama运行的Gemma 2B的智能体示例 ---
    agent_ollama_gemma = LlmAgent(
        # LiteLLM默认知道如何连接本地Ollama服务器
        model=LiteLlm(model="ollama/gemma:2b"), # Ollama的标准LiteLLM格式
        name="ollama_gemma_agent",
        instruction="您是通过Ollama本地运行的Gemma。",
        # ... 其他智能体参数
    )
    ```

### 自托管终端（如vLLM）

[vLLM](https://github.com/vllm-project/vllm)等工具可让您高效托管模型，并通常提供OpenAI兼容的API终端。

**设置**：

1. **部署模型**：使用vLLM（或类似工具）部署所选模型。记录API基础URL（例如`https://your-vllm-endpoint.run.app/v1`）。
    * *ADK工具重要提示*：部署时确保服务工具支持并启用OpenAI兼容的工具/函数调用功能。对于vLLM，可能需要使用`--enable-auto-tool-choice`等标志，并根据模型可能需要特定的`--tool-call-parser`。请参考vLLM关于工具使用的文档。
2. **认证**：确定您的终端如何处理认证（例如API密钥、Bearer令牌）。

    **集成示例**：

    ```python
    import subprocess
    from google.adk.agents import LlmAgent
    from google.adk.models.lite_llm import LiteLlm

    # --- 使用vLLM终端托管模型的智能体示例 ---

    # vLLM部署提供的终端URL
    api_base_url = "https://your-vllm-endpoint.run.app/v1"

    # 您的vLLM终端配置识别的模型名称
    model_name_at_endpoint = "hosted_vllm/google/gemma-3-4b-it" # 来自vllm_test.py的示例

    # 认证（示例：对Cloud Run部署使用gcloud身份令牌）
    # 根据您的终端安全需求进行调整
    try:
        gcloud_token = subprocess.check_output(
            ["gcloud", "auth", "print-identity-token", "-q"]
        ).decode().strip()
        auth_headers = {"Authorization": f"Bearer {gcloud_token}"}
    except Exception as e:
        print(f"警告：无法获取gcloud令牌 - {e}。终端可能未受保护或需要其他认证方式。")
        auth_headers = None # 或适当处理错误

    agent_vllm = LlmAgent(
        model=LiteLlm(
            model=model_name_at_endpoint,
            api_base=api_base_url,
            # 如果需要，传递认证头信息
            extra_headers=auth_headers
            # 或者，如果终端使用API密钥：
            # api_key="YOUR_ENDPOINT_API_KEY"
        ),
        name="vllm_agent",
        instruction="您是在自托管vLLM终端上运行的助手。",
        # ... 其他智能体参数
    )
    ```

## 使用Vertex AI上的托管及调优模型

为了获得企业级扩展性、可靠性和与Google Cloud MLOps生态系统的集成，您可以使用部署到Vertex AI终端的模型。这包括来自Model Garden的模型或您自己微调的模型。

**集成方法**：将完整的Vertex AI终端资源字符串(`projects/PROJECT_ID/locations/LOCATION/endpoints/ENDPOINT_ID`)直接传递给`LlmAgent`的`model`参数。

**Vertex AI设置（整合版）**：

确保您的环境已配置好Vertex AI：

1. **认证**：使用应用默认凭证(ADC)：

    ```shell
    gcloud auth application-default login
    ```

2. **环境变量**：设置您的项目和位置：

    ```shell
    export GOOGLE_CLOUD_PROJECT="YOUR_PROJECT_ID"
    export GOOGLE_CLOUD_LOCATION="YOUR_ENDPOINT_LOCATION" # 例如us-central1
    ```

3. **启用Vertex后端**：关键是要确保`google-genai`库指向Vertex AI：

    ```shell
    export GOOGLE_GENAI_USE_VERTEXAI=TRUE
    ```

### Model Garden部署

您可以从[Vertex AI Model Garden](https://console.cloud.google.com/vertex-ai/model-garden)部署各种开源和专有模型到终端。

**示例**：

```python
from google.adk.agents import LlmAgent
from google.genai import types # For config objects

# --- Example Agent using a Llama 3 model deployed from Model Garden ---

# Replace with your actual Vertex AI Endpoint resource name
llama3_endpoint = "projects/YOUR_PROJECT_ID/locations/us-central1/endpoints/YOUR_LLAMA3_ENDPOINT_ID"

agent_llama3_vertex = LlmAgent(
    model=llama3_endpoint,
    name="llama3_vertex_agent",
    instruction="You are a helpful assistant based on Llama 3, hosted on Vertex AI.",
    generate_content_config=types.GenerateContentConfig(max_output_tokens=2048),
    # ... other agent parameters
)
```

### 微调模型终端

部署您微调的模型（无论是基于Gemini还是Vertex AI支持的其他架构）将生成可直接使用的终端。

**示例**：

```python
from google.adk.agents import LlmAgent

# --- Example Agent using a fine-tuned Gemini model endpoint ---

# Replace with your fine-tuned model's endpoint resource name
finetuned_gemini_endpoint = "projects/YOUR_PROJECT_ID/locations/us-central1/endpoints/YOUR_FINETUNED_ENDPOINT_ID"

agent_finetuned_gemini = LlmAgent(
    model=finetuned_gemini_endpoint,
    name="finetuned_gemini_agent",
    instruction="You are a specialized assistant trained on specific data.",
    # ... other agent parameters
)
```

### Vertex AI上的第三方模型（如Anthropic Claude）

一些提供商（如Anthropic）通过Vertex AI直接提供他们的模型。

**集成方法**：使用直接模型字符串（例如`"claude-3-sonnet@20240229"`），但需要在ADK中手动注册。

**为何需要注册？** ADK的注册表会自动识别`gemini-*`字符串和标准Vertex AI终端字符串(`projects/.../endpoints/...`)，并通过`google-genai`库路由它们。对于通过Vertex AI直接使用的其他模型类型（如Claude），您必须明确告知ADK注册表哪个特定的封装类（本例中为`Claude`）知道如何处理该模型标识字符串与Vertex AI后端。

**设置**：

1. **Vertex AI环境**：确保完成整合的Vertex AI设置（ADC、环境变量、`GOOGLE_GENAI_USE_VERTEXAI=TRUE`）。
2. **安装提供商库**：安装为Vertex AI配置的必要客户端库。

    ```shell
    pip install "anthropic[vertex]"
    ```

3. **注册模型类**：在应用程序启动时添加此代码，在使用Claude模型字符串创建智能体之前：

    ```python
    # 使用Vertex AI直接通过Claude模型字符串与LlmAgent配合所需
    from google.adk.models.anthropic_llm import Claude
    from google.adk.models.registry import LLMRegistry

    LLMRegistry.register(Claude)
    ```

    **示例**：

    ```python
    from google.adk.agents import LlmAgent
    from google.adk.models.anthropic_llm import Claude # 注册所需导入
    from google.adk.models.registry import LLMRegistry # 注册所需导入
    from google.genai import types

    # --- 注册Claude类（在启动时执行一次） ---
    LLMRegistry.register(Claude)

    # --- 使用Vertex AI上Claude 3 Sonnet的智能体示例 ---

    # Vertex AI上Claude 3 Sonnet的标准模型名称
    claude_model_vertexai = "claude-3-sonnet@20240229"

    agent_claude_vertexai = LlmAgent(
        model=claude_model_vertexai, # 注册后传递直接字符串
        name="claude_vertexai_agent",
        instruction="您是在Vertex AI上由Claude 3 Sonnet驱动的助手。",
        generate_content_config=types.GenerateContentConfig(max_output_tokens=4096),
        # ... 其他智能体参数
    )
    ```