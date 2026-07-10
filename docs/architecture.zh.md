# EvoSkill 架构：新手入门指南

## 全局概览（30 秒版）

```
你提供给 EvoSkill：
  - 一份包含题目 + 正确答案的 CSV
  - 一段任务描述（"回答金融类问题"）

EvoSkill 运行一个循环：
  1. Agent 尝试答题 → 部分失败
  2. "Proposer" agent 分析失败的原因
  3. "Generator" agent 创建修复方案（一个新的 skill 文件）
  4. Agent 带着新 skill 重试 → 分数提升了吗？
  5. 是 → 保留，否 → 丢弃
  6. 重复最多 20 次，或直到卡住为止

你得到：
  - 表现最佳的 agent 配置
  - 所有发现的 skills
  - 一个展示提升幅度的分数
```

---

## 如何理解各个分层

把 EvoSkill 想象成一个**有 4 层的洋葱**。每一层都建立在内层之上。

### 第 1 层（最内层）：核心理念 —— "尝试、失败、学习、重复"

**文件**：`src/feedback_descent.py`

这是最简单的部分。它就是一个循环：

```
current_best = starting_point
history_of_failures = []

repeat:
    new_attempt = make_something_new(current_best, history_of_failures)

    is_new_better = compare(current_best, new_attempt)

    if yes:
        current_best = new_attempt
        history_of_failures = []   ← 重新开始，旧的失败不再重要
    else:
        history_of_failures.append("这次没成功，因为……")

    if too_many_failures_in_a_row:
        give up
```

**关键洞察**：一旦某个方案成功了，就忘掉所有旧的失败。为什么？因为你现在有了新的基线 —— 那些针对旧版本的失败可能已经不再相关了。

**这段代码是抽象的** —— 它不知道 agent、skill 或题目的存在。它只知道"尝试各种方案，保留更好的，记住哪些没用"。

---

### 第 2 层：循环 —— "让 Agent 变得更好"

**文件**：`src/loop/runner.py`（这是最大、最重要的文件）

这一层把抽象的"尝试、失败、学习"理念应用到 **AI agent** 上。

#### 什么是"程序（Program）"？

一个"程序" = 定义 agent 行为方式的全部内容：

- 它的 **system prompt**（告诉它如何思考的指令）
- 它的 **skills**（`.claude/skills/` 下赋予它新能力的文件）

#### 每次迭代发生了什么？

以下是一次循环迭代，逐步拆解：

**步骤 1：挑选一个父代来改进**

```
"我们应该尝试改进哪个版本的 agent？"
→ 通常是得分最高的那个（但也可以随机选，以增加多样性）
```

**步骤 2：找出哪里出了问题**

```
在一些训练题目上运行 agent
"嘿 agent，Q3 的营收是多少？"
Agent 回答："$4.2B"       ← 正确，跳过
Agent 回答："我不知道"     ← 错误，这是一个失败

收集所有失败（通常每次迭代 4-6 个）
```

**步骤 3：问 Proposer "我们该怎么办？"**

```
Proposer agent 会拿到：
  - 失败详情（agent 尝试了什么、哪里出错了）
  - 过往尝试的历史（"上次我们试了 X，没有帮助"）
  - 已有 skills 的列表

Proposer 会给出类似这样的建议：
  "Agent 在百分比计算上反复失败。
   我们应当 CREATE 一个名为 'percentage-calculator' 的新 skill，
   教它如何计算同比变化。"
```

**步骤 4：让 Generator 来构建它**

```
Generator agent 拿到 Proposer 的想法，实际写出代码。
它会创建类似这样的文件：.claude/skills/percentage-calculator/SKILL.md
```

**步骤 5：测试新版本**

```
带着新 skill，在一组独立的题目（验证集）上运行 agent
"分数上升了吗？"
```

**步骤 6：保留或丢弃**

```
如果分数提升了 → 保留！存为一个 git 分支。
如果分数没提升 → 删除它，记录"这次没成功"
如果连续 5 次迭代都没有提升 → 停止，我们卡住了
```

#### 为什么用两组题目？

- **训练集**：用于发现失败（步骤 2）。proposer 会看到这些。
- **验证集**：用于测试改进（步骤 5）。proposer 从不接触这些。

这可以防止"作弊" —— agent 可能学会应付特定的训练题目，却并没有真正变得更强。

---

### 第 3 层：支撑性基础设施

这些是让循环可靠运行的辅助系统。

#### 3a. Harness 层（`src/harness/`）

**作用**：封装各种 agent SDK（Claude Code、OpenCode，以及未来的 Goose/OpenHands），让你能运行 agent 并获得结构化结果。它与 agent 配置相分离，因此新增一个 harness 不会触碰任何配置代码。

```python
# 简化视图：
from src.harness import Agent, AgentTrace

agent = Agent(options=如何配置该agent, response_model=输出长什么样)
result = await agent.run("2+2 等于几？")

result.output        # → 解析后的答案（例如 AgentResponse，final_answer="4"）
result.total_cost_usd  # → 花了多少钱
result.duration_ms     # → 花了多长时间
```

harness 目录包含：

```
src/harness/
├── agent.py               ← Agent 类 + AgentTrace（公共接口）
├── sdk_config.py          ← 全局 SDK 开关（set_sdk("claude") / set_sdk("opencode")）
├── options_utils.py       ← build_claudecode_options() / build_opencode_options()
├── _claude_executor.py    ← Claude SDK 执行 + 响应解析
└── _opencode_executor.py  ← OpenCode SDK 执行 + 服务器管理 + 解析
```

- **重试**：若 SDK 失败，会重试 3 次，延迟递增（30s、60s、120s）
- **超时**：每次调用最多 20 分钟
- **结构化输出**：将响应强制约束为 Pydantic 模型（类似带类型的字典）
- **按项目独立的服务器**：OpenCode 作为本地 HTTP 服务器运行 —— 每个项目分配独立端口

#### 3b. Agent 配置（`src/agent_profiles/`）

**作用**：定义每个 agent 角色**做什么**（system prompt、工具、输出 schema）。从 harness 层导入以为当前激活的 SDK 构建 options。

共有 **5 个 agent 角色**，各有不同的指令和工具：

| 角色                 | 职责                                      | 输出                                                       |
| -------------------- | ----------------------------------------- | ---------------------------------------------------------- |
| **Base Agent**       | 回答实际的题目                            | `{final_answer, reasoning}`                                |
| **Skill Proposer**   | 分析失败，提议要构建什么                  | `{action: "create"/"edit", proposed_skill, justification}` |
| **Skill Generator**  | 实际写出 skill 代码                       | `{generated_skill, reasoning}`                             |
| **Prompt Proposer**  | 建议如何修改 system prompt                | `{proposed_prompt_change, justification}`                  |
| **Prompt Generator** | 重写 system prompt                        | `{optimized_prompt, reasoning}`                            |

#### 3c. Git 版本管理（`src/registry/manager.py`）

**作用**：agent 的每个版本 = 一个 git 分支。

```
program/base           ← 起始 agent
program/iter-skill-1   ← 添加第一个 skill 后
program/iter-skill-2   ← 添加第二个 skill 后（可能被丢弃）
program/iter-skill-3   ← 添加第三个 skill 后
```

每个分支包含：

- `.claude/program.yaml` —— agent 的配置
- `.claude/skills/` —— 它拥有的 skill 文件

**"frontier（前沿）"** = 得分最高的前 3 个版本，用 git tag 追踪。

为什么用 git？因为你可以：

- 回到任意版本：`git checkout program/iter-skill-1`
- 精确查看改动：`git diff program/base program/iter-skill-3`
- 永不丢失成果 —— 被丢弃的分支会被删除，但好的分支会保留

#### 3d. 缓存（`src/cache/run_cache.py`）

**作用**：在没有变化时避免重复运行同一道题。

agent 每次迭代可能要回答 50 道验证题。如果两次迭代之间 skills 没有变化，为什么还要重跑全部 50 道？缓存结果即可！

**精妙之处**：只有当"影响行为"的文件发生变化时才使缓存失效：

- Skill 文件变了 → 缓存失效，全部重跑
- 只是更新了 metadata 里的分数 → 缓存仍有效，复用结果

#### 3e. 评分（`src/evaluation/`）

**作用**：判定答案是否正确。

默认评分器使用**多容差匹配**：

- 完全匹配："4.2" = "4.2" → 满分
- 接近匹配："4.19" ≈ "4.2"（在 1% 以内）→ 部分得分
- 相差甚远："5.0" ≠ "4.2" → 不得分

针对特定任务的专用评分器：

- **SEAL-QA**：使用另一个 AI 模型来判断答案在语义上是否正确
- **LiveCodeBench**：在 Docker 容器中运行生成的代码，检查测试是否通过

---

### 第 4 层（最外层）：你如何使用它

使用 EvoSkill 有两种方式：

#### 方式 1：Python API

```python
from src import EvoSkill

result = await EvoSkill(
    dataset="questions.csv",
    task="sealqa",
    mode="skill_only",      # 发现新 skills（相对于重写 prompt）
    max_iterations=20,
).run()

print(result.best_score)        # 例如 0.85
print(result.best_program)      # 例如 "iter-skill-7"
print(result.iterations_completed)  # 例如 12
```

#### 方式 2：CLI

```bash
# 1. 初始化你的项目
evoskill init
# → 创建 .evoskill/config.toml（设置）和 .evoskill/task.md（agent 应做什么）

# 2. 编辑配置和任务描述

# 3. 运行改进循环
evoskill run
# → 展示一个实时终端表格，显示分数、发现的 skills 等

# 4. 查看发现了什么
evoskill skills      # 列出所有 skills
evoskill logs        # 查看运行历史
evoskill diff base iter-skill-5  # 比较两个版本
```

CLI 从 `.evoskill/config.toml` 加载设置：

```toml
[harness]
name = "claude"
model = "sonnet"

[evolution]
mode = "skill_only"
iterations = 20
frontier_size = 3

[dataset]
path = "data/questions.csv"
question_column = "question"
ground_truth_column = "ground_truth"

[scorer]
type = "multi_tolerance"
```

---

## 所有文件如何连接在一起

```
你
 │
 ├── Python: EvoSkill(...)          CLI: evoskill run
 │   (src/api/evoskill.py)          (src/cli/commands/run.py)
 │           │                              │
 │           └──────────┬───────────────────┘
 │                      │
 │                      ▼
 │            SelfImprovingLoop        ← 主引擎
 │            (src/loop/runner.py)        (第 2 层)
 │                      │
 │      ┌───────────────┼───────────────┐
 │      ▼               ▼               ▼
 │  Agent 配置       Git 版本管理     缓存
 │  (src/agent_      (src/registry/)  (src/cache/)
 │   profiles/)           │               │
 │      │                 │               │
 │      ▼                 ▼               ▼
 │  Harness 层        Git 分支        .cache/runs/
 │  (src/harness/)    program/*       {hash}.json
 │      │             .claude/
 │      ▼             program.yaml
 │  Claude/OpenCode   skills/
 │  (未来：Goose、
 │   OpenHands)
 │
 └── 评分 (src/evaluation/)
     └── "答案正确吗？" → 0 到 1 之间的浮点数
```

---

## 两种进化模式解析

### skill_only（默认，推荐）

system prompt 保持不变。循环发现新的 **skill 文件**。

可以这样理解："agent 的性格保持不变，但它学会了新本领。"

一个 skill 文件就是位于 `.claude/skills/{name}/SKILL.md` 的一个 markdown 文件，教会 agent 某项具体能力（比如如何计算百分比，或如何读取财务表格）。

### prompt_only

skills 保持不变。循环重写 **system prompt**。

可以这样理解："agent 的本领保持不变，但我们改变它思考的方式。"

位于 `src/agent_profiles/base_agent/prompt.txt` 的 prompt 会在每次迭代时被重写。

---

## 需要了解的关键数值

| 设置                    | 默认值  | 含义                                        |
| ---------------------- | ------- | ------------------------------------------- |
| `max_iterations`       | 20      | 最大改进尝试次数                            |
| `frontier_size`        | 3       | 保留得分最高的前 3 个版本                   |
| `no_improvement_limit` | 5       | 若连续 5 次迭代无提升则停止                 |
| `train_ratio`          | 0.18    | 18% 的数据用于发现失败                      |
| `val_ratio`            | 0.12    | 12% 的数据用于测试改进                      |
| `concurrency`          | 4       | 并行运行 4 个评估                           |
| 通过阈值               | 0.8     | 分数低于 80% = 失败                         |
| Agent 超时             | 20 分钟 | 每次 agent 调用的最大耗时                   |
| Agent 重试             | 3       | 失败的 API 调用重试 3 次                    |

---

## 总结：一段话版本

EvoSkill 接收一份题目数据集，将其拆分为训练/验证集，在训练题上运行 AI agent 以发现失败，用一个 "proposer" agent 分析这些失败并建议一个新 skill，用一个 "generator" agent 构建该 skill，在验证题上测试改进后的 agent，若分数上升则保留（存为一个 git 分支），然后重复 —— 全过程同时进行结果缓存、成本追踪，并维护一个由得分最高的前 3 个版本组成的 "frontier"。
