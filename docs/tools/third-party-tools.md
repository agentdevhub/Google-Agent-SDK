# 第三方工具集成

ADK 被设计为**高度可扩展的系统，可无缝集成来自其他 AI Agent 框架（如 CrewAI 和 LangChain）的工具**。这种互操作性至关重要，既能加快开发速度，又能复用现有工具资源。

## 1. 使用 LangChain 工具

ADK 提供 `LangchainTool` 包装器，可将 LangChain 生态系统的工具集成到您的智能体中。

### 示例：使用 LangChain 的 Tavily 工具实现网络搜索

[Tavily](https://tavily.com/) 提供的搜索 API 能返回基于实时搜索结果的答案，专为 AI 智能体等应用程序设计。

1. 遵循 [ADK 安装指南](../get-started/installation.md)完成基础配置

2. **安装依赖项：** 确保已安装必要的 LangChain 组件。例如使用 Tavily 搜索工具时需安装特定依赖：

    ```bash
    pip install langchain_community tavily-python
    ```

3. 获取 [Tavily](https://tavily.com/) API 密钥并设置为环境变量：

    ```bash
```py
--8<-- "examples/python/snippets/tools/third-party/langchain_tavily_search.py"
```
```py
--8<-- "examples/python/snippets/tools/third-party/crewai_serper_search.py"
```