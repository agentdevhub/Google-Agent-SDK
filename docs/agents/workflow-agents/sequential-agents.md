# 顺序型智能体

## `SequentialAgent`

`SequentialAgent`是一种[工作流智能体](index.md)，它会按照列表中指定的顺序依次执行其子智能体。

当您需要以固定且严格的顺序执行任务时，请使用`SequentialAgent`。

### 示例场景

* 假设您需要构建一个能总结任意网页内容的智能体，该智能体需要使用两个工具：`get_page_contents`和`summarize_page`。由于该智能体必须始终先调用`get_page_contents`再调用`summarize_page`（没有内容就无法生成摘要！），此时您应该使用`SequentialAgent`来构建这个智能体。

与其他[工作流智能体](index.md)相同，`SequentialAgent`并非由大模型驱动，因此其执行过程是完全确定性的。需要注意的是，工作流智能体仅关注执行顺序（即顺序执行），而不干涉内部逻辑——工作流智能体包含的工具或子智能体可能会使用大模型，也可能不会。

### 工作原理

当调用`SequentialAgent`的`run_async()`方法时，会执行以下操作：

1. **顺序遍历**：按照预设顺序遍历`sub_agents`列表
2. **子智能体执行**：为列表中的每个子智能体调用其`run_async()`方法

![Sequential Agent](../../assets/sequential-agent.png){: width="600"}

### 完整示例：代码开发流水线

来看一个简化的代码开发流程：

* **代码编写智能体**：基于需求文档生成初始代码的`LlmAgent`
* **代码审查智能体**：检查生成代码是否存在错误、风格问题及是否符合最佳实践的`LlmAgent`，它会接收代码编写智能体的输出
* **代码重构智能体**：根据审查意见对代码进行质量优化的`LlmAgent`

这种情况非常适合使用`SequentialAgent`：

```py
SequentialAgent(sub_agents=[CodeWriterAgent, CodeReviewerAgent, CodeRefactorerAgent])
```

这样可以严格确保代码先被编写，再被审查，最后进行重构，执行顺序完全可靠。**每个子智能体的输出会通过[`output_key`](../llm-agents.md)机制存储在状态中传递给下一个智能体**。

???+ "代码实现"

    ```py
    --8<-- "examples/python/snippets/agents/workflow-agents/sequential_agent_code_development_agent.py"
    ```