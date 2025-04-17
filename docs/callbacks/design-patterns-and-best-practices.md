# 回调的设计模式与最佳实践

回调机制为智能体生命周期提供了强大的钩子功能。本文将展示在ADK中高效运用回调的常见设计模式，并附上实现时的最佳实践指南。

## 设计模式

以下模式展示了使用回调增强或控制智能体行为的典型方法：

### 1. 防护栏与策略执行

* **模式**：在请求到达大模型或工具前进行拦截以执行规则
* **实现**：使用`before_model_callback`检查`LlmRequest`提示词，或通过`before_tool_callback`检查工具参数（`args`）。若检测到策略违规（如禁用话题、不当用语），返回预定义响应（`LlmResponse`或`dict`）以阻断操作，并可选更新`context.state`记录违规
* **示例**：`before_model_callback`检查`llm_request.contents`中的敏感关键词，若发现则返回标准"Cannot process this request"`LlmResponse`，阻止大模型调用

### 2. 动态状态管理

* **模式**：在回调中读写会话状态，使智能体行为具备上下文感知能力并在步骤间传递数据
* **实现**：访问`callback_context.state`或`tool_context.state`。修改操作（`state['key'] = value`）会被自动记录到后续`Event.actions.state_delta`中，由`SessionService`实现持久化
* **示例**：`after_tool_callback`将工具结果中的`transaction_id`保存至`tool_context.state['last_transaction_id']`。后续`before_agent_callback`可能读取`state['user_tier']`来自定义智能体问候语

### 3. 日志记录与监控

* **模式**：在特定生命周期节点添加详细日志，提升可观测性和调试能力
* **实现**：通过回调（如`before_agent_callback`、`after_tool_callback`、`after_model_callback`）打印或发送结构化日志，包含智能体名称、工具名称、调用ID及上下文/参数中的相关数据
* **示例**：记录类似`INFO: [Invocation: e-123] Before Tool: search_api - Args: {'query': 'ADK'}`的日志消息

### 4. 缓存机制

* **模式**：通过缓存结果避免冗余的大模型调用或工具执行
* **实现**：在`before_model_callback`或`before_tool_callback`中基于请求/参数生成缓存键，检查`context.state`（或外部缓存）。若命中则直接返回缓存的`LlmResponse`或结果`dict`；若未命中则允许操作执行，并在对应`after_`回调（`after_model_callback`、`after_tool_callback`）中将新结果存入缓存
* **示例**：`before_tool_callback`检查`state[f"cache:stock:{symbol}"]`中的`get_stock_price(symbol)`。若存在则返回缓存价格，否则允许API调用并由`after_tool_callback`将结果保存至状态键

### 5. 请求/响应修改

* **模式**：在数据发送至大模型/工具前或接收后立即进行修改
* **实现**：
    * `before_model_callback`：修改`llm_request`（如基于`state`添加系统指令）
    * `after_model_callback`：修改返回的`LlmResponse`（如格式化文本、过滤内容）
    * `before_tool_callback`：修改工具`args`字典
    * `after_tool_callback`：修改`tool_response`字典
* **示例**：若`context.state['lang'] == 'es'`存在，`before_model_callback`会在`llm_request.config.system_instruction`后追加"User language preference: Spanish"

### 6. 条件式步骤跳过

* **模式**：基于特定条件阻止标准操作（智能体运行、大模型调用、工具执行）
* **实现**：从`before_`回调（`Content`、`LlmResponse`、`dict`）返回值。框架会将该值视为步骤结果，跳过正常执行流程
* **示例**：`before_tool_callback`检查`tool_context.state['api_quota_exceeded']`。若`True`则返回`{'error': 'API quota exceeded'}`，阻止实际工具函数运行

### 7. 工具专属操作（鉴权与摘要控制）

* **模式**：处理工具生命周期中的专属操作，主要包括鉴权和控制大模型对工具结果的摘要生成
* **实现**：在工具回调（`before_tool_callback`、`after_tool_callback`）中使用`ToolContext`
    * **鉴权**：在`before_tool_callback`中调用`tool_context.request_credential(auth_config)`（当需要凭证但未找到时，通过`tool_context.get_auth_response`或状态检查），触发认证流程
    * **摘要控制**：设置`tool_context.actions.skip_summarization = True`可将工具原始字典输出直接传回大模型或显示，绕过默认的摘要生成步骤
* **示例**：安全API的`before_tool_callback`检查状态中的认证令牌，若缺失则调用`request_credential`；返回结构化JSON的工具可能设置`skip_summarization = True`

### 8. 产物处理

* **模式**：在智能体生命周期中保存或加载会话相关文件/大数据块
* **实现**：使用`callback_context.save_artifact`/`tool_context.save_artifact`存储数据（如生成报告、日志、中间数据），通过`load_artifact`检索已存产物。变更通过`Event.actions.artifact_delta`追踪
* **示例**："generate_report"工具的`after_tool_callback`使用`tool_context.save_artifact("report.pdf", report_part)`保存输出文件；`before_agent_callback`可能通过`callback_context.load_artifact("agent_config.json")`加载配置产物

## 回调最佳实践

* **功能聚焦**：每个回调应专注单一明确目的（如仅日志记录、仅验证）。避免构建多功能混杂的回调
* **性能考量**：回调在智能体处理循环中同步执行。避免长时间运行或阻塞操作（网络调用、重计算）。必要时卸载任务，但需注意会增加复杂度
* **优雅容错**：在回调函数中使用`try...except`块。合理记录错误并决定应中止智能体调用还是尝试恢复。避免回调错误导致整个进程崩溃
* **谨慎状态管理**：
    * 对`context.state`的读写操作需谨慎。变更在*当前*调用中立即生效，并在事件处理结束时持久化
    * 使用特定状态键而非修改宽泛结构，避免副作用
    * 考虑使用状态前缀（`State.APP_PREFIX`、`State.USER_PREFIX`、`State.TEMP_PREFIX`）提升可读性，特别是持久化`SessionService`实现时
* **幂等设计**：若回调会产生外部副作用（如外部计数器递增），尽可能设计为幂等操作（相同输入可安全重复执行），以应对框架或应用中的潜在重试
* **全面测试**：使用模拟上下文对象对回调函数进行单元测试。执行集成测试确保回调在完整智能体流程中正常工作
* **清晰表达**：为回调函数使用描述性名称。添加清晰文档字符串说明其目的、触发时机及副作用（特别是状态修改）
* **正确上下文类型**：始终使用指定的上下文类型（大模型/智能体用`CallbackContext`，工具用`ToolContext`），确保访问正确的方法和属性

通过应用这些模式与最佳实践，您可以在ADK中有效利用回调机制，构建更健壮、可观测且可定制的智能体行为。