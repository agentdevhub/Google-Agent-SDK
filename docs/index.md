---
hide:
  - toc
---

  <div class="centered-logo-text-group">
    <img src="assets/agent-development-kit.png" alt="Agent Development Kit Logo" width="100">
    <h1>Agent Development Kit</h1>
  </div>
</div>

<p style="text-align:center; font-size: 1.2em;">
  <b>一个与Gemini和Google深度集成的开源AI智能体框架</b><br/>
</p>

## 什么是Agent Development Kit？

Agent Development Kit（ADK）是一个灵活模块化的框架，用于**开发和部署AI智能体**。ADK可与主流大模型及开源生成式AI工具配合使用，其设计重点在于**与Google生态系统及Gemini模型的深度集成**。该框架既能**轻松构建基于Gemini模型和Google AI工具的简易智能体**，又能为**复杂智能体架构与编排**提供所需的控制力和结构化支持。

<div class="install-command-container">
  <p style="text-align:center;">
    Get started:
    <br/>
    <code>pip install google-adk</code>
  </p>
</div>

<p style="text-align:center;">
  <a href="get-started/quickstart/" class="md-button">快速开始</a>
  <a href="get-started/tutorial/" class="md-button">教程</a>
  <a href="http://github.com/google/adk-samples" class="md-button" target="_blank">示例智能体</a>
  <a href="api-reference/" class="md-button">API参考</a>
  <a href="contributing-guide/" class="md-button">贡献 ❤️</a>
</p>

---

## 了解更多

<div class="grid cards" markdown>

-   :material-transit-connection-variant: **Flexible Orchestration**

    ---

    Define workflows using workflow agents (`Sequential`, `Parallel`, `Loop`)
    for predictable pipelines, or leverage LLM-driven dynamic routing
    (`LlmAgent` transfer) for adaptive behavior.

    [**Learn about agents**](agents/index.md)

-   :material-graph: **Multi-Agent Architecture**

    ---

    Build modular and scalable applications by composing multiple specialized
    agents in a hierarchy. Enable complex coordination and delegation.

    [**Explore multi-agent systems**](agents/multi-agents.md)

-   :material-toolbox-outline: **Rich Tool Ecosystem**

    ---

    Equip agents with diverse capabilities: use pre-built tools (Search, Code
    Exec), create custom functions, integrate 3rd-party libraries (LangChain,
    CrewAI), or even use other agents as tools.

    [**Browse tools**](tools/index.md)

-   :material-rocket-launch-outline: **Deployment Ready**

    ---

    Containerize and deploy your agents anywhere – run locally, scale with
    Vertex AI Agent Engine, or integrate into custom infrastructure using Cloud
    Run or Docker.

    [**Deploy agents**](deploy/index.md)

-   :material-clipboard-check-outline: **Built-in Evaluation**

    ---

    Systematically assess agent performance by evaluating both the final
    response quality and the step-by-step execution trajectory against
    predefined test cases.

    [**Evaluate agents**](evaluate/index.md)

-   :material-console-line: **Building Responsible Agents**

    ---

    Learn how to building powerful and trustworthy agents by implementing
    responsible AI patterns and best practices into your agent's design.

    [**Responsible agents**](guides/responsible-agents.md)

</div>

!!! 预览版说明

    本功能适用《通用服务条款》中"预GA产品条款"的规定，详见
    [服务专项条款](https://cloud.google.com/terms/service-terms#1)。
    预GA功能按"现状"提供，可能仅获得有限支持。更多信息请参阅
    [产品发布阶段说明](https://cloud.google.com/products#product-launch-stages)。