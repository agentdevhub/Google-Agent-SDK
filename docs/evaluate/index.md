# 为何需要评估智能体

在传统软件开发中，单元测试和集成测试能确保代码按预期运行并在变更后保持稳定。这些测试提供明确的"通过/失败"信号，指导后续开发。然而大模型智能体引入了不确定性，使得传统测试方法不再适用。

由于模型的概率特性，确定性的"通过/失败"断言往往不适合评估智能体表现。我们需要对最终输出和智能体轨迹（即解决问题的步骤序列）进行定性评估。这包括评估智能体的决策质量、推理过程以及最终结果。

虽然建立评估体系看似需要额外工作，但自动化评估的投入能快速获得回报。如果您希望超越原型阶段，这绝对是最佳实践。

![intro_components.png](../assets/evaluate_agent.png)

## 评估前的准备工作

在自动化评估前，需明确目标和成功标准：

* **定义成功标准**：您的智能体达成何种结果才算成功？  
* **识别关键任务**：智能体必须完成哪些核心任务？  
* **选择相关指标**：追踪哪些指标来衡量性能？

这些考量将指导评估场景的创建，并有效监控实际部署中的智能体行为。

## 评估什么？

要跨越概念验证与生产级AI智能体之间的鸿沟，必须建立稳健的自动化评估框架。与生成式模型评估不同（仅关注最终输出），智能体评估需要深入理解决策过程。评估可分为两部分：

1. **评估轨迹和工具使用**：分析智能体解决问题的步骤，包括工具选择、策略及方法效率  
2. **评估最终响应**：评判最终输出的质量、相关性和准确性

轨迹就是智能体返回结果前执行的操作步骤列表。我们可以将其与预期步骤列表进行比对。

### 评估轨迹和工具使用

在响应用户前，智能体通常会执行一系列动作（称为"轨迹"）。它可能比对用户输入和会话历史来消除术语歧义，或查询政策文档、搜索知识库、调用API创建工单。我们称这些动作为"轨迹"。评估表现需要将实际轨迹与预期（理想）轨迹对比，从而发现流程中的错误和低效。预期轨迹代表基准事实——我们期望智能体执行的步骤序列。

例如：

```py
// Trajectory evaluation will compare
expected_steps = ["determine_intent", "use_tool", "review_results", "report_generation"]
actual_steps = ["determine_intent", "use_tool", "review_results", "report_generation"]
```

存在多种基于基准事实的轨迹评估方法：

1. **精确匹配**：要求与理想轨迹完全一致  
2. **顺序匹配**：要求动作顺序正确，允许额外动作  
3. **任意顺序匹配**：只需包含正确动作，顺序不限，允许额外动作  
4. **精确率**：衡量预测动作的相关性/正确性  
5. **召回率**：衡量预测包含多少必要动作  
6. **单一工具检查**：验证是否包含特定动作

选择合适指标取决于智能体的具体需求和目标。例如高风险场景可能需要精确匹配，而灵活场景可能只需顺序或任意顺序匹配。

## ADK的评估机制

ADK提供两种方法，基于预设数据集和评估标准来评测智能体表现。虽然概念相似，但二者处理的数据量不同，通常决定了各自的适用场景。

### 方法一：使用测试文件

该方法需创建独立的测试文件，每个文件代表一次简单的智能体-模型交互（会话）。最适合活跃开发阶段，相当于单元测试。这些测试设计用于快速执行，应关注简单会话场景。每个测试文件包含单个会话（可能含多轮交互）。每轮交互包括：

* `query:` 用户查询语句  
* `expected_tool_use`：智能体为正确响应应调用的工具 `query`  
* `expected_intermediate_agent_responses`：该字段包含智能体生成最终答案过程中产生的自然语言响应。这些响应常见于多智能体系统，当根智能体依赖子智能体完成任务时产生。虽然通常不直接面向终端用户，但这些中间响应对开发者极具价值，能揭示智能体的推理路径，帮助验证其是否遵循正确步骤生成最终响应  
* `reference`：模型的预期最终响应

文件名可任意指定（如 `evaluation.test.json`）。框架仅检测 `.test.json` 后缀，前缀不受限制。以下是示例测试文件：

```json
[
  {
    "query": "hi",
    "expected_tool_use": [],
    "expected_intermediate_agent_responses": [],
    "reference": "Hello! What can I do for you?\n"
  },
  {
    "query": "roll a die for me",
    "expected_tool_use": [
      {
        "tool_name": "roll_die",
        "tool_input": {
          "sides": 6
        }
      }
    ],
    "expected_intermediate_agent_responses": [],
  },
  {
    "query": "what's the time now?",
    "expected_tool_use": [],
    "expected_intermediate_agent_responses": [],
    "reference": "I'm sorry, I cannot access real-time information, including the current time. My capabilities are limited to rolling dice and checking prime numbers.\n"
  }
]
```

测试文件可组织到文件夹中。可选地，文件夹可包含 `test_config.json` 文件来指定评估标准。

### 方法二：使用评估集文件

评估集方法使用专用数据集（称为"evalset"）来评估智能体-模型交互。与测试文件类似，但评估集可包含多个可能冗长的会话，非常适合模拟复杂的多轮对话。因其能呈现复杂会话，评估集特别适合集成测试。由于测试规模较大，通常执行频率低于单元测试。

评估集文件包含多个"eval"，每个代表独立会话。每个eval含一至多轮"turns"，包括用户查询、预期工具使用、预期中间响应和参考响应。这些字段含义与测试文件方法相同。每个eval有唯一名称，且包含关联的初始会话状态。

手动创建评估集较复杂，因此提供了UI工具帮助捕获相关会话并轻松转换为评估集条目。下方将详述网页UI使用方法。以下是包含两个会话的评估集示例：

```json
[
  {
    "name": "roll_16_sided_dice_and_then_check_if_6151953_is_prime",
    "data": [
      {
        "query": "What can you do?",
        "expected_tool_use": [],
        "expected_intermediate_agent_responses": [],
        "reference": "I can roll dice of different sizes and check if a number is prime. I can also use multiple tools in parallel.\n"
      },
      {
        "query": "Roll a 16 sided dice for me",
        "expected_tool_use": [
          {
            "tool_name": "roll_die",
            "tool_input": {
              "sides": 16
            }
          }
        ],
        "expected_intermediate_agent_responses": [],
        "reference": "I rolled a 16 sided die and got 13.\n"
      },
      {
        "query": "Is 6151953  a prime number?",
        "expected_tool_use": [
          {
            "tool_name": "check_prime",
            "tool_input": {
              "nums": [
                6151953
              ]
            }
          }
        ],
        "expected_intermediate_agent_responses": [],
        "reference": "No, 6151953 is not a prime number.\n"
      }
    ],
    "initial_session": {
      "state": {},
      "app_name": "hello_world",
      "user_id": "user"
    }
  },
  {
    "name": "roll_17_sided_dice_twice",
    "data": [
      {
        "query": "What can you do?",
        "expected_tool_use": [],
        "expected_intermediate_agent_responses": [],
        "reference": "I can roll dice of different sizes and check if a number is prime. I can also use multiple tools in parallel.\n"
      },
      {
        "query": "Roll a 17 sided dice twice for me",
        "expected_tool_use": [
          {
            "tool_name": "roll_die",
            "tool_input": {
              "sides": 17
            }
          },
          {
            "tool_name": "roll_die",
            "tool_input": {
              "sides": 17
            }
          }
        ],
        "expected_intermediate_agent_responses": [],
        "reference": "I have rolled a 17 sided die twice. The first roll was 13 and the second roll was 4.\n"
      }
    ],
    "initial_session": {
      "state": {},
      "app_name": "hello_world",
      "user_id": "user"
    }
  }
]
```

### 评估标准

评估标准定义了如何根据评估集衡量智能体表现。支持以下指标：

* `tool_trajectory_avg_score`：该指标将评估期间智能体的实际工具使用与 `expected_tool_use` 字段定义的预期使用进行比对。每个匹配的工具使用步骤得1分，不匹配得0分。最终得分是这些匹配的平均值，代表工具使用轨迹的准确度  
* `response_match_score`：该指标比较智能体的最终自然语言响应与 `reference` 字段存储的预期响应。我们使用[ROUGE](https://en.wikipedia.org/wiki/ROUGE_\(metric\))指标计算两者相似度

若未提供评估标准，则使用以下默认配置：

* `tool_trajectory_avg_score`：默认为1.0，要求工具使用轨迹100%匹配  
* `response_match_score`：默认为0.8，允许智能体自然语言响应存在小幅误差

以下是 `test_config.json` 文件指定自定义评估标准的示例：

```json
{
  "criteria": {
    "tool_trajectory_avg_score": 1.0,
    "response_match_score": 0.8
  }
}
```

## 如何使用ADK运行评估

开发者可通过以下方式使用ADK评估智能体：

1. **网页UI（**`adk web`**）**：通过交互式网页界面评估  
2. **编程方式（**`pytest`**）**：使用 `pytest` 和测试文件将评估集成到测试流程  
3. **命令行界面（**`adk eval`**）**：直接从命令行对现有评估集文件运行评估

### 1. `adk web` - 通过网页UI运行评估

网页UI提供交互式评估方式，并可生成评估数据集。

网页UI评估步骤：

1. 运行以下命令启动服务器：`bash adk web samples_for_testing`  
2. 在网页界面中：  
    * 选择智能体（如 `hello_world`）  
    * 与智能体交互创建要保存为测试用例的会话  
    * 点击界面右侧**"Eval标签"**  
    * 如有现有评估集则选择，或点击**"Create new eval set"**新建。为评估集起情境化名称，选择新建的评估集  
    * 点击**"Add current session"**将当前会话保存为评估集条目。需为此eval命名（建议情境化名称）  
    * 创建后，新eval将显示在评估集文件列表中。可运行全部或选择特定eval执行评估  
    * 每个eval状态将显示在UI中

### 2. `pytest` - 编程方式运行测试

您也可以使用 **`pytest`** 将测试文件作为集成测试的一部分运行。

#### 示例命令

```shell
pytest tests/integration/
```

#### 测试代码示例

以下是运行单个测试文件的 `pytest` 测试用例示例：

```py
def test_with_single_test_file():
    """Test the agent's basic ability via a session file."""
    AgentEvaluator.evaluate(
        agent_module="tests.integration.fixture.home_automation_agent",
        eval_dataset="tests/integration/fixture/home_automation_agent/simple_test.test.json",
    )
```

此方法允许将智能体评估集成到CI/CD流程或大型测试套件中。如需为测试指定初始会话状态，可将会话详情存入文件并传递给 `AgentEvaluator.evaluate` 方法。

以下是会话JSON文件示例：

```json
{
  "id": "test_id",
  "app_name": "trip_planner_agent",
  "user_id": "test_user",
  "state": {
    "origin": "San Francisco",
    "interests": "Moutains, Hikes",
    "range": "1000 miles",
    "cities": ""


  },
  "events": [],
  "last_update_time": 1741218714.258285
}
```

对应示例代码如下：

```py
def test_with_single_test_file():
    """Test the agent's basic ability via a session file."""
    AgentEvaluator.evaluate(
        agent_module="tests.integration.fixture.trip_planner_agent",
        eval_dataset="tests/integration/fixture/trip_planner_agent/simple_test.test.json",
        initial_session_file="tests/integration/fixture/trip_planner_agent/initial.session.json"
    )
```

### 3. `adk eval` - 通过CLI运行评估

也可通过命令行界面(CLI)对评估集文件运行评估。这与UI执行的评估相同，但有助于自动化（例如可将此命令加入常规构建验证流程）。

命令如下：

```shell
adk eval \
    <AGENT_MODULE_FILE_PATH> \
    <EVAL_SET_FILE_PATH> \
    [--config_file_path=<PATH_TO_TEST_JSON_CONFIG_FILE>] \
    [--print_detailed_results]
```

例如：

```shell
adk eval \
    samples_for_testing/hello_world \
    samples_for_testing/hello_world/hello_world_eval_set_001.evalset.json
```

各命令行参数说明：

* `AGENT_MODULE_FILE_PATH`：指向包含名为"agent"模块的 `init.py` 文件路径。"agent"模块需包含 `root_agent`  
* `EVAL_SET_FILE_PATH`：评估文件路径。可指定一个或多个评估集文件路径。默认运行每个文件的所有eval。如需仅运行特定eval，先创建逗号分隔的eval名称列表，然后以冒号 `:` 为分隔符附加在文件名后  
  例如：`sample_eval_set_file.json:eval_1,eval_2,eval_3`  
  `This will only run eval_1, eval_2 and eval_3 from sample_eval_set_file.json`  
* `CONFIG_FILE_PATH`：配置文件路径  
* `PRINT_DETAILED_RESULTS`：在控制台打印详细结果