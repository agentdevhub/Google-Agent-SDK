# 工作流代理

本节将介绍"*工作流代理*"——**一类专门用于控制子代理执行流程的特殊代理**。

工作流代理是ADK中专为**编排子代理执行流程**设计的特殊组件。它们的主要职责是管理其他代理的*运行方式*和*触发时机*，从而定义整个流程的控制逻辑。

与[LLM代理](../llm-agents.md)不同（后者依赖大模型进行动态推理和决策），工作流代理完全基于**预定义逻辑**运行。它们根据自身类型（如顺序型、并行型、循环型）确定执行顺序，而无需通过大模型进行编排决策，因此能产生**确定且可预测的执行模式**。

ADK提供三种核心工作流代理类型，每种类型对应不同的执行模式：

<div class="grid cards" markdown>

- :material-console-line: **Sequential Agents**

    ---

    Executes sub-agents one after another, in **sequence**.

    [:octicons-arrow-right-24: Learn more](sequential-agents.md)

- :material-console-line: **Loop Agents**

    ---

    **Repeatedly** executes its sub-agents until a specific termination condition is met.

    [:octicons-arrow-right-24: Learn more](loop-agents.md)

- :material-console-line: **Parallel Agents**

    ---

    Executes multiple sub-agents in **parallel**.

    [:octicons-arrow-right-24: Learn more](parallel-agents.md)

</div>

## 为何使用工作流代理？

当您需要精确控制一系列任务或代理的执行顺序时，工作流代理不可或缺。它们能提供：

* **确定性**：执行流程完全由代理类型和配置保证
* **可靠性**：确保任务始终按既定顺序或模式运行
* **结构化**：通过清晰的流程控制结构组合多个代理，构建复杂流程

虽然工作流代理以确定性方式管理流程，但它所编排的子代理可以是任何类型——包括智能的`LlmAgent`实例。这种特性让您能将结构化流程控制与基于大模型的灵活任务执行相结合。