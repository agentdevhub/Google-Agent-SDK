# 记忆模块：`MemoryService`实现的长时知识存储

我们已经了解`Session`如何追踪*单次持续对话*中的历史记录（`events`）和临时数据（`state`）。但当智能体需要回忆*过往*对话内容或访问外部知识库时，就需要**长时知识**和**`MemoryService`**的概念发挥作用。

可以这样理解：

* **`Session` / `State`：** 如同单次对话中的短期记忆  
* **长时知识（`MemoryService`）：** 如同可检索的档案库或知识库，智能体可以查阅其中可能包含的多次历史对话内容或其他来源的信息

## `MemoryService`的核心作用

`BaseMemoryService`定义了管理这种可检索长时知识存储的接口，其主要职责包括：

1. **信息摄入（%%PH_92a94c2%%）：** 获取（通常已完成的）`Session`内容，并将相关信息添加至长时知识存储  
2. **信息检索（`search_memory`）：** 允许智能体（通常通过`Tool`）查询知识库，根据搜索条件返回相关片段或上下文

## `MemoryService`实现方案

ADK提供多种长时知识存储实现方式：

1. **`InMemoryMemoryService`**  

    * **工作原理：** 在应用内存中存储会话信息，通过基础关键词匹配实现搜索  
    * **持久性：** 无。**应用重启后所有存储知识都将丢失**  
    * **依赖项：** 无需额外配置  
    * **适用场景：** 原型开发、简单测试、仅需基础关键词召回且无需持久化的场景  

    ```py
    from google.adk.memory import InMemoryMemoryService
    memory_service = InMemoryMemoryService()
    ```

2. **`VertexAiRagMemoryService`**  

    * **工作原理：** 利用Google Cloud的Vertex AI RAG（检索增强生成）服务。将会话数据摄入指定的RAG语料库，使用强大的语义搜索能力进行检索  
    * **持久性：** 有。知识持久存储在配置的Vertex AI RAG语料库中  
    * **依赖项：** 需要Google Cloud项目、适当权限、必要SDK（`pip install google-adk[vertexai]`）以及预配置的Vertex AI RAG语料库资源名称/ID  
    * **适用场景：** 需要可扩展、持久化且语义相关检索的生产级应用，特别是部署在Google Cloud环境时  

    ```py
    # 需要：pip install google-adk[vertexai]
    # 以及GCP配置、RAG语料库和认证
    from google.adk.memory import VertexAiRagMemoryService

    # RAG语料库名称或ID
    RAG_CORPUS_RESOURCE_NAME = "projects/your-gcp-project-id/locations/us-central1/ragCorpora/your-corpus-id"
    # 可选的检索配置
    SIMILARITY_TOP_K = 5
    VECTOR_DISTANCE_THRESHOLD = 0.7

    memory_service = VertexAiRagMemoryService(
        rag_corpus=RAG_CORPUS_RESOURCE_NAME,
        similarity_top_k=SIMILARITY_TOP_K,
        vector_distance_threshold=VECTOR_DISTANCE_THRESHOLD
    )
    ```

## 实际工作流程

典型工作流程包含以下步骤：

1. **会话交互：** 用户通过`Session`与智能体交互，由`SessionService`管理。事件被添加，状态可能更新  
2. **记忆存储：** 当会话被认为完成或产生重要信息时，应用调用`memory_service.add_session_to_memory(session)`。这会从会话事件中提取相关信息并存入长时知识存储（内存字典或RAG语料库）  
3. **后续查询：** 在*不同*（或相同）会话中，用户可能提出需要历史上下文的问题（如"我们上周关于项目X讨论了什么？"）  
4. **调用记忆工具：** 配备记忆检索工具（如内置`load_memory`工具）的智能体识别需要历史上下文，调用工具并提交搜索查询（如"上周项目X讨论"）  
5. **执行搜索：** 工具内部调用`memory_service.search_memory(app_name, user_id, query)`  
6. **返回结果：** `MemoryService`搜索存储（使用关键词匹配或语义搜索），返回相关片段作为`SearchMemoryResponse`（包含`MemoryResult`对象列表，每个对象可能包含相关历史会话事件）  
7. **结果应用：** 工具将结果返回智能体（通常作为上下文或函数响应）。智能体利用检索到的信息生成最终回答  

## 示例：记忆的添加与检索

以下示例使用`InMemory`服务演示基础流程：

???+ "完整代码"

    ```py
    import asyncio
    from google.adk.agents import LlmAgent
    from google.adk.sessions import InMemorySessionService, Session
    from google.adk.memory import InMemoryMemoryService # 导入MemoryService
    from google.adk.runners import Runner
    from google.adk.tools import load_memory # 查询记忆的工具
    from google.genai.types import Content, Part

    # --- 常量 ---
    APP_NAME = "memory_example_app"
    USER_ID = "mem_user"
    MODEL = "gemini-2.0-flash" # 使用有效模型

    # --- 智能体定义 ---
    # 智能体1：用于捕获信息的简单智能体
    info_capture_agent = LlmAgent(
        model=MODEL,
        name="InfoCaptureAgent",
        instruction="确认用户的陈述",
        # output_key="captured_info" # 可选保存到状态
    )

    # 智能体2：可使用记忆的智能体
    memory_recall_agent = LlmAgent(
        model=MODEL,
        name="MemoryRecallAgent",
        instruction="回答用户问题。若答案可能在历史对话中，使用'load_memory'工具",
        tools=[load_memory] # 为智能体提供工具
    )

    # --- 服务与运行器 ---
    session_service = InMemorySessionService()
    memory_service = InMemoryMemoryService() # 演示使用内存存储

    runner = Runner(
        # 初始使用信息捕获智能体
        agent=info_capture_agent,
        app_name=APP_NAME,
        session_service=session_service,
        memory_service=memory_service # 为Runner提供记忆服务
    )

    # --- 场景演示 ---

    # 第一轮：在会话中捕获信息
    print("--- 第一轮：信息捕获 ---")
    session1_id = "session_info"
    session1 = session_service.create_session(APP_NAME, USER_ID, session1_id)
    user_input1 = Content(parts=[Part(text="我最喜欢的项目是Project Alpha。")])

    # 运行智能体
    final_response_text = "(无最终响应)"
    for event in runner.run(USER_ID, session1_id, user_input1):
        if event.is_final_response() and event.content and event.content.parts:
            final_response_text = event.content.parts[0].text
    print(f"智能体1响应：{final_response_text}")

    # 获取已完成的会话
    completed_session1 = session_service.get_session(APP_NAME, USER_ID, session1_id)

    # 将会话内容添加至记忆服务
    print("\n--- 将会话1添加至记忆 ---")
    memory_service.add_session_to_memory(completed_session1)
    print("会话已存入记忆")

    # 第二轮：在*新*（或相同）会话中提出需要记忆的问题
    print("\n--- 第二轮：信息回忆 ---")
    session2_id = "session_recall" # 可使用相同或不同会话ID
    session2 = session_service.create_session(APP_NAME, USER_ID, session2_id)

    # 将运行器切换至回忆智能体
    runner.agent = memory_recall_agent
    user_input2 = Content(parts=[Part(text="我最喜欢的项目是什么？")])

    # 运行回忆智能体
    print("运行MemoryRecallAgent...")
    final_response_text_2 = "(无最终响应)"
    for event in runner.run(USER_ID, session2_id, user_input2):
        print(f"  事件：{event.author} - 类型：{'文本' if event.content and event.content.parts and event.content.parts[0].text else ''}"
            f"{'函数调用' if event.get_function_calls() else ''}"
            f"{'函数响应' if event.get_function_responses() else ''}")
        if event.is_final_response() and event.content and event.content.parts:
            final_response_text_2 = event.content.parts[0].text
            print(f"智能体2最终响应：{final_response_text_2}")
            break # 最终响应后停止

    # 第二轮预期事件序列：
    # 1. 用户发送"我最喜欢的项目是什么？"
    # 2. 智能体（大模型）决定调用`load_memory`工具，查询如"favorite project"
    # 3. Runner执行`load_memory`工具，调用`memory_service.search_memory`
    # 4. `InMemoryMemoryService`从session1找到相关文本（"我最喜欢的项目是Project Alpha。"）
    # 5. 工具在FunctionResponse事件中返回该文本
    # 6. 智能体（大模型）接收函数响应，处理检索到的文本
    # 7. 智能体生成最终答案（如"您最喜欢的项目是Project Alpha。"）
    ```