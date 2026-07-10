# Agent 配置与 Schema

## Schema、Agent 配置与 harness 三者的关系

- **`src/schemas/`** = 输出模板。定义每个 agent 在响应中必须填写哪些字段。
- **`src/agent_profiles/`** = 岗位职责 + 配置。定义每个 agent 是谁、拥有哪些工具、必须使用哪个 schema。
- **`src/harness/`** = SDK 执行层。Agent 配置从这里导入，为当前激活的 SDK 构建 options。详见 `docs/sdk-support.md`。

每个 agent 配置都精确指向一个 schema 作为它的 `response_model`。

---

## 6 个 Schema

### 1. `AgentResponse` — Base Agent 的输出
```
final_answer: str    ← "营收为 $4.2B"
reasoning: str       ← "我在 Q3 报告第 12 页找到了这个数据……"
```
被所有任务型 agent 使用（base、dabstep、sealqa、livecodebench）。

### 2. `ProposerResponse` — 统一路由器（遗留）
```
optimize_prompt_or_skill: "prompt" 或 "skill"
proposed_skill_or_prompt: str
justification: str
```
最初"一个 proposer 决定一切"的方案。目前仍保留，但较新的代码使用专门化的 proposer。

### 3. `SkillProposerResponse` — Skill Proposer 的输出
```
action: "create" 或 "edit"     ← 新建一个 skill 还是修复已有的？
target_skill: str | None       ← 如果是编辑，是哪个 skill？
proposed_skill: str            ← 描述要构建/修改什么
justification: str             ← 为什么这能解决失败
related_iterations: list[str]  ← 例如 ["iter-4", "iter-9"]
```
最丰富的 schema。追踪谱系，让 proposer 能从过往失败中学习。

### 4. `ToolGeneratorResponse` — Skill Generator 的输出
```
generated_skill: str   ← 实际的 skill 代码/markdown
reasoning: str         ← 为什么这样构建
```

### 5. `PromptProposerResponse` — Prompt Proposer 的输出
```
proposed_prompt_change: str   ← 描述要改什么
justification: str            ← 为什么
```

### 6. `PromptGeneratorResponse` — Prompt Generator 的输出
```
optimized_prompt: str   ← 重写后的完整 system prompt
reasoning: str          ← 改了什么、为什么
```

---

## 两条流水线

```
Agent 答错了题
         │
         ▼
    ┌─────────────────────────────────────────────────┐
    │  SKILL 路径（skill_only 模式）                  │
    │                                                 │
    │  Skill Proposer ──► SkillProposerResponse       │
    │    "创建一个 percentage-calculator skill"        │
    │         │                                       │
    │         ▼                                       │
    │  Skill Generator ──► ToolGeneratorResponse      │
    │    写入 .claude/skills/percentage-calculator/    │
    └─────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────┐
    │  PROMPT 路径（prompt_only 模式）                │
    │                                                 │
    │  Prompt Proposer ──► PromptProposerResponse     │
    │    "Agent 应始终校验计算结果"                    │
    │         │                                       │
    │         ▼                                       │
    │  Prompt Generator ──► PromptGeneratorResponse   │
    │    重写 src/agent_profiles/base_agent/          │
    │    prompt.txt                                   │
    └─────────────────────────────────────────────────┘
```

---

## 9 个 Agent 配置

### "Worker" 类 agent —— 全套工具 + 写权限

它们负责回答基准测试题目。全部使用 `AgentResponse` schema。

| Agent | 自定义 Prompt | 特殊配置 |
|-------|--------------|----------------|
| **Base Agent** | `prompt.txt`（"你是一位专家分析师……"） | `permission_mode='acceptEdits'`、`max_buffer_size=10MB`，通过 `setting_sources=["user", "project"]` 从磁盘加载 skills |
| **DabStep Agent** | `prompt.txt`（文件缺失！） | 与 base 相同，但只有单个 `data_dir` |
| **SealQA Agent** | `prompt.txt`（文件缺失！） | 与 base 相同，无 data_dirs |
| **LiveCodeBench Agent** | 无自定义 prompt（使用 Claude 默认） | 唯一同时支持 Claude + OpenCode SDK 的 agent |

### "Meta" 类 agent —— 有限工具，负责分析/改进

| Agent | 职责 | 关键 prompt 规则 | 工具 |
|-------|-------------|-----------------|-------|
| **Proposer**（遗留） | 在 skill 与 prompt 路径间路由 | "如果是 WHAT 步骤 → skill。如果是 HOW 思考 → prompt" | 只读（8 个工具） |
| **Skill Proposer** | 分析失败，提议 skill | "必须先使用 Brainstorming skill。检查已有 skills。参照被 DISCARDED 的迭代。" | 只读（8 个工具） |
| **Skill Generator** | 构建实际的 skill 文件 | "先读 `.claude/skills/skill-creator/SKILL.md`。skill 要简洁。" | 全套工具（11 个）+ `permission_mode='acceptEdits'` |
| **Prompt Proposer** | 分析失败，提议 prompt 改动 | "仅当问题在于 HOW 思考而非 WHAT 步骤时才提议" | 只读（8 个工具） |
| **Prompt Generator** | 重写 system prompt | "反过拟合规则。Prompt 指导 HOW 而非 WHAT。这能帮助 10 个不同任务吗？" | 只读（8 个工具） |

---

## 工具清单

**全套工具箱（11 个工具）** —— 供 worker 类 agent + skill generator 使用：
```
Read, Write, Bash, Glob, Grep, Edit, WebFetch, WebSearch, TodoWrite, BashOutput, Skill
```

**只读工具箱（8 个工具）** —— 供 proposer + prompt generator 使用：
```
Read, Bash, Glob, Grep, WebFetch, WebSearch, TodoWrite, BashOutput
```

关键区别：没有 `Write`、`Edit` 或 `Skill`。Proposer 只能分析，不能修改文件。

---

## 工厂模式

所有配置工厂都从 `src.harness` 导入，并根据当前激活的 SDK 分支：

```python
from src.harness import build_claudecode_options, build_opencode_options, is_claude_sdk

def get_my_agent_options(model=None, project_root=None):
    if is_claude_sdk():
        return build_claudecode_options(system=..., schema=..., tools=..., ...)
    return build_opencode_options(system=..., schema=..., tools=..., ...)
```

有三种调用模式：

### 1. 惰性单例（meta 类 agent）
```python
# 在导入时使用当前激活的 SDK 求值一次
skill_proposer_options = get_skill_proposer_options()
```
使用者：skill_proposer、skill_generator、prompt_proposer、prompt_generator。

### 2. 直接工厂函数（任务型 agent）
```python
def get_base_agent_options(model=None, data_dirs=None):
    prompt_text = PROMPT_FILE.read_text()  # 每次都从磁盘读取
    return _build_base_agent_options(prompt_text, model=model, data_dirs=data_dirs)
```
使用者：base_agent、dabstep_agent、sealqa_agent。

### 3. 返回工厂的工厂（用于 Agent[T]）
```python
def make_base_agent_options(model=None):
    def factory():
        return get_base_agent_options(model=model)
    return factory  # 返回函数本身，而非结果
```
这个函数会被传入 `Agent(options=factory, ...)`。Agent 在每次 `.run()` 时调用 `factory()`，从而每次都拿到新鲜的 options。

---

## 关键设计细节

1. **Proposer 被刻意设为只读。** 没有 Write 或 Edit 工具。它们只应分析和提议。

2. **Skill Generator 拥有 `Skill` 工具。** 它会先读取 `skill-creator` skill 来学习正确格式，再构建新 skill。

3. **Base Agent 在运行时从磁盘读取自己的 prompt。** 这对 `prompt_only` 模式至关重要 —— Prompt Generator 重写 `prompt.txt`，下一次运行自动生效。

4. **Prompt Generator 有反过拟合规则。** "错误示范：`use np.std(ddof=1)`" vs "正确示范：为你的样本类型选择合适的方法。"

5. **Skill Proposer 必须使用 Brainstorming skill。** 强制在定案前给出 2-3 种备选方案。

6. **`related_iterations` 追踪谱系。** proposer 引用过往被丢弃的尝试，以避免重复犯错。

7. **worker 类 agent 上的 `setting_sources=["user", "project"]`** 正是让发现的 skills 能从 `.claude/skills/` 加载的关键。
