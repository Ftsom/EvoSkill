# EvoSkill 技术文档

> EvoSkill 是由 Sentient Labs 提出的自动化技能发现与 Agent 进化框架，论文标题为《EvoSkill: Automated Skill Discovery for Multi-Agent Systems》。
>
> 其核心思想是将 **Skill 文件**类比为生物体的**可遗传基因**，将**失败驱动的改进提案**类比为**自然选择的适应性突变**，将**Frontier（精英种群）**类比为**种群中的优势个体**。在这种视角下，Agent 的能力不再是静态配置，而是一组"外部可进化的技能文档"，由专门的 Proposer Agent 分析失败模式并提出改进，Generator Agent 实施变异，Evaluator 做适应度选择，Frontier 维护精英种群。
>
> 这一思路将此前"手工编写 Skill/Prompt"的工程实践，转化为有明确闭环、可量化、可复现的自动化进化流程。

---

## 1. 概述

EvoSkill 是一个**不修改模型权重**的 Agent 能力自动进化框架。它通过类似进化算法的迭代循环（选择、变异、评估、淘汰），自动发现和优化 **Skill 文档**（或 System Prompt），使 Agent 在特定任务上的表现持续提升。

### 1.1 核心类比

| EvoSkill 概念 | 进化算法类比 | 说明 |
|---|---|---|
| Skill 文档 / System Prompt | 基因 (genome) | 被优化的对象，一段 Markdown 指令文件 |
| Base Agent 执行任务 | 个体表现 (phenotype expression) | 用当前 skill 执行任务，得到预测结果 |
| Proposer 分析失败 | 选择压力 (selection pressure) | 分析失败模式，识别能力缺口 |
| Generator 生成 Skill | 突变 (mutation) | 创建或修改 skill 文件，产生新变体 |
| Eval Gate | 适应度评估 (fitness evaluation) | 在验证集上评估新变体的表现 |
| Frontier | 精英种群 (elite population) | 维护 top-N 最优 Agent 配置 |
| Feedback History | 进化记忆 (evolutionary memory) | 记录历史提案和结果，指导后续突变方向 |
| Git Branch | 谱系 (lineage) | 每个程序变体存储为独立 git 分支 |

### 1.2 多 Agent 角色架构

| 角色 | 职责 | 输出 |
|---|---|---|
| **Base Agent** | 用当前 skill 执行实际任务 | `AgentResponse { final_answer, reasoning }` |
| **Skill Proposer** | 分析失败 case，提出新 skill 或修改方案 | `SkillProposerResponse { action, target_skill, proposed_skill, justification }` |
| **Skill Generator** | 实现 proposer 的提案，写入 SKILL.md 文件 | `ToolGeneratorResponse { generated_skill, reasoning }` |
| **Prompt Proposer** | 分析失败，提出 prompt 修改方案 | `PromptProposerResponse { proposed_prompt_change, justification }` |
| **Prompt Generator** | 实现 prompt 修改，输出优化后的 prompt | `PromptGeneratorResponse { optimized_prompt, reasoning }` |

进化模式通过 `evolution_mode` 配置切换：`skill_only`（默认）只进化 Skill 文件，`prompt_only` 只进化 System Prompt。

---

## 2. 核心机制

EvoSkill 的自动进化闭环由五个阶段组成，它们共同构成一个反馈驱动的迭代优化循环。

### 2.1 Base Agent 执行：失败样本采集

每一迭代从训练池中**按类目轮询采样**（round-robin across categories）若干条题目，用当前最优 Agent（含已有 skills）执行任务。与简单记录"答对/答错"不同，EvoSkill 保留完整的**执行轨迹**——Agent 的推理过程、工具调用序列、中间结果、最终答案等原始信息都会被保存。

评分阈值为 0.8：得分 ≥ 0.8 视为通过，< 0.8 收集为失败样本。失败样本附带完整 trace，作为 Proposer 的直接输入——如果 trace 不完整，Proposer 就看不到 Agent 实际是"卡在哪一步"失败的。

**类目轮询采样**确保覆盖所有题目类别，避免优化偏向某一类别。每个类目维护独立的采样偏移量，循环取样，保证数据多样性。

### 2.2 Proposer：失败分析与改进提案

Proposer 接收多个失败 case 的完整轨迹，进行结构化分析：

1. **轨迹审查**：逐步检查 Agent 的执行过程（做了什么、在哪卡住、什么推理模式）
2. **差距分析**：对比 Agent 答案与标准答案（缺了什么信息、犯了什么推理错误）
3. **现有 Skill 检查**：审查已有 skill 列表，判断是否有 skill 本该覆盖却没生效
4. **反馈历史参照**：检查历史中被 DISCARDED 的类似提案，避免重复犯错

Proposer 的输出决策：
- **action="create"**：没有现有 skill 覆盖此能力 → 提议创建新 skill
- **action="edit"**：已有 skill 本该防止此失败但未生效 → 提议修改现有 skill

**渐进式上下文截断**（Progressive Truncation）机制确保在 LLM 上下文限制内仍能正常工作：
- Level 0：完整上下文（60K head + 60K tail）
- Level 1：中度截断（20K head + 10K tail，反馈限制 20 行，最多 3 个失败样本）
- Level 2：激进截断（5K head + 2K tail，反馈限制 5 行，最多 2 个失败样本）

如果所有截断级别都失败，还可启用**单失败回退**：只取最短的一个失败样本重试。

### 2.3 Generator：实施变异

Generator 根据 Proposer 的提案，执行具体的文件操作：

**Skill 模式**：在 `.claude/skills/<skill-name>/SKILL.md` 路径写入或修改 Skill 文件。每个 Skill 文件遵循标准格式：

```yaml
---
name: answer-unit-preservation
description: Preserve required output units for arithmetic answers.
---

## 规则内容
...
```

**Prompt 模式**：读取当前 prompt（`parent_config.system_prompt`），根据提案生成优化后的完整 prompt 文本，写入 `src/agent_profiles/base_agent/prompt.txt`。子程序分支命名为 `iter-prompt-{N}`。

### 2.4 Evaluator：验证集评估

新变体创建后，在**独立验证集**上完整评估表现。评估并发执行（默认 4 路并发），每个 Agent 调用有 17 分钟硬超时。

**评分机制**（默认 `multi_tolerance`）：
- 加权平均多个容差级别（0%、1%、2.5%、5%、10%）
- 严格容差权重更高：`weight = 1 / (1 + 20 × tolerance)`
- 支持模糊数字匹配、单位检测（trillion/billion/million）、文本重叠检查
- 也支持精确匹配、LLM 评判、自定义脚本评分

### 2.5 Frontier：精英种群管理

Frontier 维护 top-N（默认 3）个最优 Agent 配置，通过 git tag 标记：

```
程序分支：  program/base, program/iter-skill-1, program/iter-skill-2, ...
Frontier 标签：  frontier/base, frontier/iter-skill-2, ...
```

**加入规则**：
- Frontier 未满 → 无条件加入
- Frontier 已满且新分数 > 最差成员分数 → 替换最差成员
- 否则 → 丢弃（git 分支删除）

**父代选择策略**：
- `best`（默认）：总是选得分最高的
- `random`：从 frontier 中均匀随机选
- `round_robin`：按排名循环选取

---

## 3. 启动流程

### 3.1 入口

```
CLI: evoskill run
Python API: EvoSkill(task="base", model="sonnet").run_sync()
```

执行命令示例：

```bash
# CLI 方式
evoskill run --continue --verbose

# Python API
from src.api import EvoSkill
result = await EvoSkill(
    task="sealqa",
    model="sonnet",
    mode="skill_only",
    max_iterations=20,
    frontier_size=3,
).run()
```

### 3.2 初始化步骤

```
1. load_config(config_path)
   ├── 加载 .evoskill/config.toml
   ├── 解析 harness / evolution / dataset / scorer 配置
   └── 设置 SDK（claude / opencode / codex / goose / openhands）

2. 构建 LoopAgents
   ├── base = Agent(base_agent_options, AgentResponse)
   ├── skill_proposer = Agent(skill_proposer_options, SkillProposerResponse)
   ├── prompt_proposer = Agent(prompt_proposer_options, PromptProposerResponse)
   ├── skill_generator = Agent(skill_generator_options, ToolGeneratorResponse)
   └── prompt_generator = Agent(prompt_generator_options, PromptGeneratorResponse)

3. 初始化 ProgramManager
   └── 基于 git 的程序版本管理器

4. 加载数据集
   ├── load_dataset() → 读取 CSV 或 Harbor 数据
   └── stratified_split() → 按 category 分层划分 train/val

5. SelfImprovingLoop.__init__()
   ├── 配置 round-robin 采样状态
   ├── 初始化 RunCache（缓存避免重复评估）
   ├── 加载 checkpoint（如有，支持断点续跑）
   └── 设置 feedback_history 路径
```

### 3.3 配置解析示例

以 `.evoskill/config.toml` 为例：

```toml
[harness]
name = "claude"              # 使用的 Agent 运行时
model = "sonnet"             # 模型别名或完整标识
data_dirs = []               # Agent 可访问的额外目录
timeout_seconds = 1200       # 单次 Agent 执行超时
max_retries = 3              # 失败重试次数

[evolution]
mode = "skill_only"          # 进化模式
iterations = 20              # 最大迭代次数
frontier_size = 3            # Frontier 容量
concurrency = 4              # 并发评估数
no_improvement_limit = 5     # 早停阈值

[dataset]
source = "csv"               # 数据源类型
path = "data/questions.csv"  # 数据集路径
question_column = "question"
ground_truth_column = "ground_truth"
category_column = "category" # 可选，分层采样用
train_ratio = 0.18           # 训练集比例
val_ratio = 0.12             # 验证集比例

[scorer]
type = "multi_tolerance"     # 评分方式
```

---

## 4. 主循环流程

```
┌─ 基线评估 ──────────────────────────────────────────────────────────┐
│ _ensure_base_program():                                              │
│   创建 program/base 分支 → 在验证集上评估 → 加入 Frontier            │
│   得到 base_score 作为后续比较基准                                     │
└────────────────────────────────────────────────────────────────────┘

for iteration in 1..max_iterations:
    ├── ① 选择父代：从 Frontier 中按策略选择 parent
    │   └── manager.switch_to(parent)  → git checkout program/{parent}
    │
    ├── ② 采样测试：Round-Robin 按类目采样 N 条训练题
    │   ├── 每个类目取 samples_per_category 条
    │   └── 循环偏移保证不重复
    │
    ├── ③ 并发执行：Base Agent 批量执行任务
    │   ├── asyncio.gather(*[agent.run(q) for q in samples])
    │   ├── 评分：scorer(question, predicted, ground_truth)
    │   │   └── score < 0.8 → 收集为失败样本
    │   └── 若全部通过（0 failures）→ continue 跳过本迭代剩余步骤
    │       （不计入 no_improvement，不影响早停）
    │
    ├── ④ 失败分析 + 变异（_mutate_with_fallback）：
    │   ├── Proposer 分析失败模式 → 输出提案
    │   ├── 若 Proposer 失败（含渐进截断重试后仍失败）：
    │   │   └── mutation_result = None → no_improvement_count += 1
    │   │       → 跳过步骤 ⑤⑥⑦，直接到 ⑧
    │   ├── 创建子程序分支 program/iter-skill-{N}
    │   ├── Generator 写入/修改 Skill 文件
    │   └── git commit 保存变更
    │
    ├── ⑤ 评估子代：在验证集上评估新变体
    │   └── evaluate_agent_parallel(agent, val_data, max_concurrent=concurrency)
    │
    ├── ⑥ Frontier 更新：
    │   ├── frontier 未满 → 直接加入 → outcome="kept" or "improved"
    │   ├── score > worst_in_frontier → 替换进入 → outcome="kept" or "improved"
    │   │   └── added=True → no_improvement_count = 0
    │   └── 否则 → 丢弃（manager.discard）→ outcome="discarded"
    │       └── added=False → no_improvement_count += 1
    │
    ├── ⑦ 反馈记录（仅在变异成功时执行）：
    │   └── append_feedback(child_name, proposal, justification,
    │         outcome, score, parent_score, active_skills)
    │
    ├── ⑧ 早停检查：
    │   └── no_improvement_count ≥ no_improvement_limit → break
    │
    └── ⑨ 保存 checkpoint

┌─ 最终结果 ──────────────────────────────────────────────────────────┐
│ LoopResult:                                                           │
│   frontier = [(program_name, score), ...]                            │
│   best_program = "iter-skill-5"                                      │
│   best_score = 0.7834                                                │
│   iterations_completed = 12                                           │
│   total_cost_usd = 12.45                                             │
└────────────────────────────────────────────────────────────────────┘
```

---

## 5. 单步详解

### 5.1 Base Agent 执行（采样与失败收集）

**目的**：用当前最优 Agent 配置执行若干训练样本，收集失败案例。

**代码位置**：`src/loop/runner.py` (L278-L329)

**流程**：

```
1. Round-Robin 类目采样:
   ├── categories_per_batch 个类目（默认 3）
   ├── 每个类目取 samples_per_category 条（默认 2）
   └── 维护 per-category offset 避免重复

2. 并发执行:
   asyncio.gather(*[agent.run(question) for question in test_samples])

3. 评分与筛选:
   for trace, (question, answer, category) in zip(traces, test_samples):
       agent_answer = trace.output.final_answer
       score = scorer(question, agent_answer, answer)
       if score < 0.8:
           failures.append((trace, agent_answer, answer, category))

4. 输出:
   failures = [(trace, agent_answer, ground_truth, category), ...]
```

**Base Agent 配置**：
- 工具集：Read, Write, Bash, Glob, Grep, Edit, WebFetch, WebSearch, TodoWrite, BashOutput, Skill
- 输出格式：`{ final_answer: str, reasoning: str }`
- Prompt：从 `.evoskill/task.md` 加载的任务描述

### 5.2 Proposer（失败分析与提案）

**目的**：分析失败 case 的执行轨迹，提出结构化的改进方案。

**代码位置**：`src/loop/helpers.py` (L16-L113), `src/agent_profiles/skill_proposer/`

**流程**：

```
1. 构建 Proposer Query:
   ├── 现有 Skills 列表（.claude/skills/ 下所有 SKILL.md）
   ├── Task Constraints（来自 .evoskill/task.md）
   ├── 反馈历史（feedback_history.md 内容）
   └── 失败样本详情:
       ├── 每个失败 case 的完整 trace summary
       ├── Agent 的回答 vs 标准答案
       └── 所属类目

2. Skill Proposer 分析:
   ├── 强制使用 Brainstorming 技能探索 2-3 种方案
   ├── 审查已有 skills 是否应该覆盖此失败
   ├── 参照反馈历史中被 DISCARDED 的提案
   └── 选择最简可行方案（YAGNI 原则）

3. 输出决策:
   {
     "action": "create" | "edit",
     "target_skill": "existing-skill-name" | null,
     "proposed_skill": "详细描述需要创建/修改什么",
     "justification": "为什么这个改进能解决识别到的问题",
     "related_iterations": ["iter-4", "iter-9"]
   }
```

**Proposer Prompt 的关键约束**：
- 必须先检查现有 skills，避免重复创建
- 必须参照反馈历史，避免重复失败的提案
- 提案必须跨 case 通用，不针对单个样本
- 优先 edit 已有 skill，而非创建新的

### 5.3 Generator（实施变异）

**目的**：将 Proposer 的高层提案转化为具体的 Skill 文件。

**代码位置**：`src/loop/runner.py` (L520-L603), `src/agent_profiles/skill_generator/`

**两种操作**：

#### 创建新 Skill (action="create")

```
1. 接收 proposer 的描述和 justification
2. 在 .claude/skills/{skill-name}/ 目录下创建 SKILL.md
3. 文件格式:
   ---
   name: answer-unit-preservation
   description: Preserve required output units for arithmetic answers.
   ---
   
   ## 规则正文
   （具体的可复用规则和示例）

4. git add + commit
```

#### 编辑已有 Skill (action="edit")

```
1. 读取 .claude/skills/{target_skill}/SKILL.md
2. 根据 proposer 的修改描述进行编辑
3. 保留原有相关内容，追加或修改目标段落
4. git add + commit
```

**Generator 的约束**：
- 必须使用 YAML frontmatter（name + description 必填）
- name 必须与目录名一致，全小写连字符
- 正文简洁具体，包含可复用规则
- 可选包含 1-3 个短示例

### 5.4 Evaluator（验证集评估）

**目的**：在独立验证集上衡量新变体的实际表现。

**代码位置**：`src/evaluation/evaluate.py`, `src/evaluation/reward.py`

**流程**：

```
1. 转换验证数据为 (question, answer) 格式

2. 并发评估:
   evaluate_agent_parallel(agent, qa_data, max_concurrent=concurrency)
   ├── Semaphore 控制并发度
   ├── 每条 17 分钟硬超时
   ├── 支持 RunCache 缓存（基于 git tree hash）
   └── 超时/错误 → 该题 0 分

3. 计算总分:
   score = sum(scorer(q, pred, gt) for each result) / total_count
```

**评分系统（`multi_tolerance`）**：

```python
TOLERANCE_LEVELS = [0.05, 0.01, 0.1, 0.0, 0.025]  # 顺序不影响结果（加权平均）

for tol in TOLERANCE_LEVELS:
    weight = 1.0 / (1.0 + 20.0 * tol)  # 严格容差权重更高
    score += weight * score_answer(ground_truth, predicted, tol)
    # score_answer 内部调用 fuzzy_match_answer

final_score = weighted_sum / weight_total
```

`fuzzy_match_answer` 的处理逻辑：
- **数字匹配**：提取数字 + 上下文，检测单位（trillion/billion/million/thousand），归一化比较
- **多数字答案**：所有 GT 数字必须在预测中找到匹配
- **文本匹配**：大小写不敏感，去括号，子串包含检查
- **混合匹配**：数字匹配成功后还需检查文本重叠

### 5.5 Frontier 更新（精英选择）

**目的**：维护 top-N 最优程序，实现优胜劣汰。

**代码位置**：`src/registry/manager.py` (L378-L430)

**流程**：

```
1. 保存分数到子程序的 program.yaml:
   git checkout program/{child}
   config.with_score(score) → 写入文件 → commit

2. 判断是否入选 Frontier:
   scored = get_frontier_with_scores()  # 当前 frontier 成员及分数
   
   if len(scored) < max_size:
       → mark_frontier(child)     # 未满，直接加入
       → return True
   
   worst_name, worst_score = scored[-1]
   if score > worst_score:
       → unmark_frontier(worst)   # 替换最差
       → mark_frontier(child)
       → return True
   
   → return False                  # 不够格，丢弃

3. 丢弃操作:
   manager.discard(child_name)     # 删除 git 分支
```

**Git Tag 管理**：
- 加入：`git tag frontier/{name}` 在对应 branch HEAD
- 移除：`git tag -d frontier/{name}`
- 查询：`git tag -l "frontier/*"`

---

## 6. 反馈机制

### 6.1 Feedback History（进化记忆）

**位置**：`.evoskill/feedback_history.md`

**目的**：让 Proposer 能够学习历史经验——什么有效、什么无效。

每次迭代结束后追加一条记录：

```markdown
## iter-skill-3
**Proposal**: Create a 'date-format-parser' skill that handles...
**Justification**: The trace shows agent failed to parse ISO dates...
**Outcome**: IMPROVED (score: 0.6234 (+0.0512))
**Active Skills**: answer-unit-preservation, date-format-parser
```

**记录的关键字段**：
- `Proposal`：提议的内容
- `Justification`：为什么这么提议
- `Outcome`：结果（IMPROVED / KEPT / DISCARDED）
- `Score`：评分及与父代的差值
- `Active Skills`：评估时活跃的 skill 列表

> 注：`IMPROVED` = 加入 Frontier 且超过父代分数；`KEPT` = 加入 Frontier 但未超过父代；`DISCARDED` = 未加入 Frontier，分支被删除。

**Proposer 如何使用**：
- 看到 DISCARDED 的提案 → 避免重复提出类似方案，或解释自己的方案有何不同
- 看到 IMPROVED 的提案 → 理解什么方向有效
- 看到 Active Skills → 了解当前 Agent 已有的能力

### 6.2 断点续跑

**Checkpoint 文件**：`.evoskill/loop_checkpoint.json`

每个迭代结束时保存：
- 当前迭代编号
- Round-robin 采样偏移状态
- 类目内采样偏移量

`evoskill run --continue` 时：
1. 读取 checkpoint → 恢复精确的采样状态
2. 保留 feedback_history → 不丢失历史经验
3. 保留 Frontier → 从当前最优继续

---

## 7. 数据流架构

```
                        Self-Improving Loop (每迭代)
                        ===========================

             ┌───────────────────────────────────────────────────┐
             │                                                   │
 Frontier ──►│ SELECT PARENT → SAMPLE → BASE AGENT → COLLECT   │
 (top-N)     │   (strategy)    (round-    (execute)   FAILURES  │
             │                  robin)                           │
             │                                                   │
             │ FAILURES ──► PROPOSER ──► GENERATOR ──► COMMIT   │
             │              (analyze)    (write SKILL)  (git)    │
             │                                                   │
             │ EVALUATE (val set) ──► UPDATE FRONTIER            │
             │  (parallel scoring)     (replace worst or add)    │
             │                                                   │
             │ RECORD FEEDBACK ──► CHECKPOINT                    │
             │                                                   │
             └───────────────────────────────────────────────────┘

                              │
                              ▼
                     Git Branch Structure
                     ====================
                     
             main (用户代码, 不动)
             program/base (基线)
             program/iter-skill-1 (迭代 1)
             program/iter-skill-2 (迭代 2)
             ...
             frontier/base (tag)
             frontier/iter-skill-5 (tag, 当前最优)
```

---

## 8. 状态与产物结构

```
.evoskill/
├── config.toml                  # 项目配置
├── task.md                      # 任务描述（Agent 的 system prompt 来源）
├── feedback_history.md          # 进化记忆（Proposer 参考）
├── loop_checkpoint.json         # 断点续跑状态
├── state.json                   # 运行状态快照
├── data/                        # 数据集
├── logs/                        # 运行日志
├── reports/                     # 评估报告
└── skills/                      # 发现的 skill 备份

.claude/
├── program.yaml                 # 当前程序配置（prompt, tools, score）
└── skills/                      # 所有已发现的 Skills
    ├── answer-unit-preservation/
    │   └── SKILL.md
    ├── date-format-parser/
    │   └── SKILL.md
    └── ...

.cache/runs/                     # 评估结果缓存（避免重复 Agent 调用）
```

**program.yaml 示例**：

```yaml
name: iter-skill-5
parent: program/iter-skill-3
generation: 5
system_prompt:
  content: "你是一位专业的中文文案创作者..."
allowed_tools:
  - Read
  - Write
  - Bash
  - Skill
  - ...
metadata:
  score: 0.7834
  created_at: "2026-07-01T10:30:00"
```

---

## 9. 支持的 Agent 运行时

EvoSkill 通过统一的 `Agent` 抽象层支持多种编码 Agent：

| 运行时 | SDK | 模型格式 | 特点 |
|---|---|---|---|
| **Claude Code** | claude-agent-sdk | `claude-sonnet-4-6` | 原生结构化输出 |
| **OpenCode** | opencode-ai | `provider/model` | 多提供商，持久化 HTTP 服务器 |
| **Codex CLI** | openai codex SDK | `gpt-5`, `o3` | 线程执行，通过 symlink 发现 skills |
| **Goose** | subprocess CLI | `provider/model` | Recipe YAML 格式 |
| **OpenHands** | litellm | `fireworks_ai/...` | Fallback JSON 提取 |
| **Harbor** | 容器化 | — | 内置验证器的基准测试框架 |

**Agent 统一接口**（`src/harness/agent.py`）：
- 超时：20 分钟/次尝试
- 重试：最多 3 次，指数退避（30s → 60s → 120s）
- 结构化输出：所有 SDK 产出统一的 `AgentTrace[T]` 结构

---

## 10. 关键配置参数速查

| 参数 | 位置 | 默认值 | 影响 |
|---|---|---|---|
| `evolution.mode` | config.toml | skill_only | 进化维度（skill / prompt） |
| `evolution.iterations` | config.toml | 7 | 最大迭代次数（Python API 默认 20） |
| `evolution.frontier_size` | config.toml | 3 | Frontier 容量 |
| `evolution.concurrency` | config.toml | 4 | 并发评估数 |
| `evolution.no_improvement_limit` | config.toml | 5 | 早停阈值 |
| `evolution.failure_samples` | config.toml | 3 | 每迭代测试样本数 |
| `dataset.train_ratio` | config.toml | 0.18 | 训练集划分比例 |
| `dataset.val_ratio` | config.toml | 0.12 | 验证集划分比例 |
| `harness.timeout_seconds` | config.toml | 1200 | Agent 执行超时 |
| `harness.max_retries` | config.toml | 3 | 失败重试次数 |
| `scorer.type` | config.toml | multi_tolerance | 评分方式 |
| `LoopConfig.categories_per_batch` | 代码 | 3 | 每迭代采样类目数 |
| `LoopConfig.samples_per_category` | 代码 | 2 | 每类目采样条数 |
| `LoopConfig.selection_strategy` | 代码 | best | 父代选择策略 |
| `LoopConfig.proposer_max_truncation_level` | 代码 | 2 | 最大上下文截断级别 |

---

## 11. 评分器类型

| 类型 | 描述 | 适用场景 |
|---|---|---|
| `multi_tolerance` | 多容差加权匹配（数字+文本） | 通用 QA（默认） |
| `exact` | 大小写不敏感精确匹配 | 选择题、固定格式答案 |
| `llm` | LLM-as-Judge 按 rubric 评分 | 开放式生成、创意写作 |
| `script` | 自定义脚本评分 | 代码执行、特殊格式 |
| `harbor` | Harbor 内置验证器 | 容器化基准测试 |

**LLM 评分器配置示例**：

```toml
[scorer]
type = "llm"
rubric = "根据以下标准评分（0.0~1.0）：1) 内容是否符合要求；2) 质量是否流畅；3) 约束是否满足。"
model = "claude-sonnet-4-6"
provider = "anthropic"    # anthropic / openai / google / openrouter / fireworks
```

---

## 12. 与相关工作对比

### 方法定位

EvoSkill 属于 **Agent Skill Discovery / Agent Self-Improvement** 方向，与 GEPA、DSPy、TextGrad、SkillOpt 等系统解决相同问题：提升 Agent 任务表现。核心循环相似——执行→评估→分析→改进→验证。

### 关键差异

| 维度 | SkillOpt | EvoSkill |
|------|----------|----------|
| **优化对象** | 单个 Skill 文档的文本内容 | Agent 的**完整配置**：多个 Skill 文件 + System Prompt |
| **优化粒度** | 原子 patch 操作（append/replace/delete） | 整个 Skill 文件的创建/修改 |
| **版本管理** | 文件快照（skill_v0001.md, skill_v0002.md） | **Git 分支 + Tag**：完整的程序谱系、可回溯、可 diff |
| **种群策略** | 单一最优 skill（eval gate 只保留 best） | **Frontier（精英种群）**：维护 top-N，支持多样性 |
| **Agent 架构** | 双模型（Optimizer + Target） | **多角色 Agent**（Proposer + Generator + Base + Evaluator） |
| **改进来源** | 只从训练 batch 的 rollout 学习 | 从失败轨迹 + **反馈历史** + 已有 skills 联合学习 |
| **跨迭代记忆** | Slow Update + Meta Skill | **Feedback History**（markdown 日志，Proposer 全部可读） |
| **Agent 兼容性** | 专用 rollout 系统 | **跨 Agent 可移植**：Claude/OpenCode/Codex/Goose/OpenHands |
| **Skill 可复用性** | 训练产物绑定特定环境 | **跨 Agent、跨模型、跨任务可迁移** |
| **验证机制** | 必须严格超越 best_score 才接受 | Frontier 机制，只需超过最差成员即可存活 |
| **学习率控制** | edit_budget + cosine scheduler | 无显式学习率，靠 Proposer 自主判断修改幅度 |
| **可恢复性** | runtime_state.json 断点续训 | Git branch + checkpoint + feedback history |

### 本质区别

1. **SkillOpt 是"Skill 的梯度下降"**：把单个 Skill 文档当作参数空间中的一个点，通过结构化 patch 做增量更新，强调收敛性和稳定性。

2. **EvoSkill 是"Agent 的进化算法"**：维护一个由多个完整 Agent 配置组成的种群（Frontier），每次产生新变体，通过适应度选择决定存活。强调多样性和发现能力。

3. **互补而非替代**：SkillOpt 适合深度优化单个已知 Skill；EvoSkill 适合从零开始自动发现 Agent 需要哪些 Skills。

### 适用场景对比

| 场景 | 推荐方法 |
|------|----------|
| 已有明确的 Skill 需要持续打磨 | SkillOpt（精细 patch + 学习率衰减） |
| 不确定 Agent 需要哪些能力，需要自动发现 | EvoSkill（自动发现 + Frontier 多样性） |
| 需要跨 Agent 可移植的 Skill 产物 | EvoSkill（标准 SKILL.md 格式） |
| 数据量大，需要类似 SGD 的系统性训练 | SkillOpt（epoch/batch/lr 完整框架） |
| 快速迭代少量 case，需要即时反馈 | EvoSkill（few-shot 失败分析 + 即时变异） |
| 需要完整的程序谱系和可审计历史 | EvoSkill（git-based lineage） |
