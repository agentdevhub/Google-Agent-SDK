# 部署您的智能体

使用 ADK 完成智能体的构建和测试后，下一步就是将其部署到生产环境，使其能够被访问、查询并与其它应用程序集成。部署过程会将智能体从本地开发环境迁移至可扩展且可靠的运行环境。

<img src="../assets/deploy-agent.png" alt="部署智能体">

## 部署选项

根据生产就绪需求或定制化灵活性的不同，您可以将 ADK 智能体部署到多种环境中：

### Vertex AI 的 Agent Engine 服务

[Agent Engine](agent-engine.md) 是 Google Cloud 上全托管的自动扩展服务，专为部署、管理和扩展基于 ADK 等框架构建的 AI 智能体而设计。

了解更多关于[将智能体部署至 Vertex AI Agent Engine](agent-engine.md) 的信息。

### Cloud Run

[Cloud Run](https://cloud.google.com/run) 是 Google Cloud 上托管的自动扩展计算平台，支持您以容器化应用的形式运行智能体。

了解更多关于[将智能体部署至 Cloud Run](cloud-run.md) 的信息。