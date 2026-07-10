# Harness 层 —— 详细指南

`src/harness/` 包负责处理**如何与各种 agent SDK 对话**。它完全不了解每个 agent 具体做什么（那是 `agent_profiles/` 的职责）。本文档解释每一个文件、每一个函数、每一项设计决策。

## 文件概览

```
src/harness/
├── __init__.py              — 包导出（对外暴露给外部导入的内容）
├── agent.py                 — Agent[T] + AgentTrace[T]（公共接口）
├── sdk_config.py            — 全局 SDK 开关（"claude" / "opencode"）
├── options_utils.py         — options 构建器 + 路径/工具/权限辅助函数
├── _claude_executor.py      — Claude SDK：启动进程、解析响应
└── _opencode_executor.py    — OpenCode SDK：管理服务器、发送查询、解析响应
```

executor 文件的下划线前缀表示"内部" —— 它们由 `agent.py` 调用，不被外部代码直接导入。

---

## `sdk_config.py` —— 全局开关

最简单的文件。一个模块级变量，系统其余部分都会检查它。

```python
_current_sdk: SDKType = "claude"     # 默认值

set_sdk("opencode")      # 全局切换
get_sdk()                 # → "opencode"
is_claude_sdk()           # → False
is_opencode_sdk()         # → True
```

**谁调用 `set_sdk()`？**
- CLI：`src/cli/commands/run.py` 读取 `config.toml` → `set_sdk(cfg.harness.name)`
- 脚本：`scripts/run_loop.py` 传入 `--sdk` 参数
- 测试：在 fixture 中 `set_sdk("claude")`

**谁检查它？**
- `options_utils.py:build_options()` —— 路由到正确的构建器
- `agent.py:Agent._execute_query()` —— 选择正确的 executor
- `agent.py:Agent.run()` —— 选择正确的响应解析器

**为什么用全局状态？** 因为 SDK 的选择会影响一次运行中的每个 agent —— 你不会在同一个循环里混用 Claude 和 OpenCode。相比在每个函数里层层传递参数，全局开关更简单。

---

## `options_utils.py` —— 构建 Agent Options

这个文件回答："给定 system prompt、schema 和 tools，我该如何构建 SDK 所需的配置对象？"

### `resolve_project_root(project_root=None) → Path`

从 `cwd` 向上查找 `.evoskill/` 或 `.git/` 来定位仓库根目录。若提供了 `project_root`，则直接使用它。

```
/Users/me/work/EvoSkill/src/loop/runner.py
    → 向上查找 → 找到 /Users/me/work/EvoSkill/.git
    → 返回 Path("/Users/me/work/EvoSkill")
```

使用者：每个 options 构建器（用于设置 agent 的 `cwd`）、CLI、API。

### `resolve_data_dirs(project_root, data_dirs=None) → list[str]`

将相对数据目录路径转换为绝对路径，相对于项目根目录解析。

```python
resolve_data_dirs("/Users/me/EvoSkill", ["data/treasury"])
# → ["/Users/me/EvoSkill/data/treasury"]

resolve_data_dirs("/Users/me/EvoSkill", ["/absolute/path"])
# → ["/absolute/path"]（已是绝对路径，保持不变）
```

### `build_options(*, system, schema, tools, ...) → Any`

**所有 9 个 agent 配置都调用的主入口。** 根据 `get_sdk()` 路由到正确的 SDK 专用构建器。

```python
# 一个配置会这样调用：
build_options(
    system="你是一位专家分析师……",
    schema=AgentResponse.model_json_schema(),
    tools=["Read", "Write", "Bash", ...],
    model="sonnet",
    setting_sources=["user", "project"],   # Claude 专用，在 OpenCode 上被忽略
    permission_mode="acceptEdits",          # Claude 专用，在 OpenCode 上被忽略
)

# 内部：
if sdk == "claude":  → build_claudecode_options(system=..., ..., setting_sources=..., permission_mode=...)
if sdk == "opencode": → build_opencode_options(system=..., ...)  # 额外参数被丢弃
```

**为什么 Claude 专用的额外参数会被静默忽略：** OpenCode 没有 `permission_mode` 或 `setting_sources` 的概念。与其强迫每个配置都了解这一点，不如让 `build_options` 始终接受它们，仅在相关时才转发。

### `build_claudecode_options(*, system, schema, tools, ...) → ClaudeAgentOptions`

构建 `claude-agent-sdk` 所需的 `ClaudeAgentOptions` 对象：

```python
ClaudeAgentOptions(
    system_prompt = {"type": "preset", "preset": "claude_code", "append": system},
    output_format = {"type": "json_schema", "schema": schema},
    allowed_tools = ["Read", "Write", "Bash", ...],
    cwd = "/Users/me/EvoSkill",
    # 可选（仅当非 None 时传入）：
    setting_sources = ["user", "project"],
    permission_mode = "acceptEdits",
    max_buffer_size = 10485760,
    add_dirs = ["/path/to/data"],
)
```

关键细节：
- `system_prompt` 始终以 `"claude_code"` preset 为基础。自定义 prompt 通过 `"append"` 追加。如果 `system` 是空字符串（livecodebench），则不添加 `"append"` 键。
- `ClaudeAgentOptions` 是在**函数内部**导入的，而非文件顶部。这意味着即使未安装 `claude-agent-sdk`，模块也能加载。
- 可选 kwargs 仅当非 `None` 时才传入 —— 避免发送可能覆盖 SDK 行为的默认值。
- `model` 在构造之后通过 `options.model = model` 设置（SDK API 要求如此）。

### `build_opencode_options(*, system, schema, tools, ...) → dict`

为 OpenCode SDK 构建一个普通字典：

```python
{
    "system": "你是一位专家分析师……\n\n额外可访问的数据目录……",
    "format": {"type": "json_schema", "schema": {...}},
    "tools": {"read": True, "write": True, "bash": True, ...},
    "mode": "build",
    "provider_id": "anthropic",
    "model_id": "claude-sonnet-4-6",
    "cwd": "/Users/me/EvoSkill",
    "add_dirs": ["/path/to/data"],
}
```

关键细节：
- **工具名映射**：Claude 使用 PascalCase（`"Read"`），OpenCode 使用小写（`"read"`）。`to_opencode_tools()` 辅助函数负责转换。`"BashOutput"` 映射为 `None`（OpenCode 不支持）。
- **模型字符串拆分**：`"anthropic/claude-sonnet-4-6"` → `provider_id="anthropic"`、`model_id="claude-sonnet-4-6"`。由 `split_opencode_model()` 完成。
- **数据目录注入**：若提供了 `data_dirs`，它们会作为一条备注追加到 system prompt 中（因为 OpenCode 没有用于 prompt 可见性的原生 `add_dirs` 机制）。
- **权限自动配置**：`ensure_opencode_project_permissions()` 在项目根目录写入一个 `opencode.json` 文件，授予对数据目录的读取权限。没有它，OpenCode 会拒绝读取项目外的文件。

### `ensure_opencode_project_permissions(project_root, data_dirs=None)`

自动创建/更新 `opencode.json` 以允许文件访问：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "external_directory": {
      "/path/to/data": "allow",
      "/path/to/data/**": "allow"
    }
  }
}
```

仅在以下情况运行：
- 有需要配置的 data_dirs
- 尚不存在 `opencode.jsonc`（用户的手动配置优先）
- 权限确实需要更新

### `to_opencode_tools(tools) → dict[str, bool]`

将 Claude 工具名映射为 OpenCode 的对应名称：

```python
to_opencode_tools(["Read", "Write", "BashOutput", "Skill"])
# → {"read": True, "write": True, "skill": True}
# 注意：BashOutput → None（被丢弃）
```

### `split_opencode_model(model) → (provider_id, model_id)`

```python
split_opencode_model("anthropic/claude-sonnet-4-6")
# → ("anthropic", "claude-sonnet-4-6")

split_opencode_model("sonnet")
# → ("anthropic", "sonnet")  # 默认 provider

split_opencode_model(None)
# → ("anthropic", "claude-sonnet-4-6")  # 默认模型
```

---

## `agent.py` —— 公共接口

这个文件定义了所有人都要用的两样东西：`AgentTrace[T]`（产出物）和 `Agent[T]`（输入物）。

### `OptionsProvider` 类型

你可以作为 `options` 传给 `Agent()` 的内容：

```python
# 选项 1：静态 ClaudeAgentOptions 对象
Agent(options=my_claude_options, response_model=AgentResponse)

# 选项 2：静态字典（OpenCode）
Agent(options={"system": "...", "tools": {...}}, response_model=AgentResponse)

# 选项 3：工厂函数（每次 .run() 时重新调用）
Agent(options=lambda: build_options(system=...), response_model=AgentResponse)
```

工厂模式（选项 3）被 `base_agent` 使用，让它在每次运行时重新从磁盘读取 `prompt.txt` —— 这对 `prompt_only` 进化模式至关重要，因为 Prompt Generator 会在迭代之间重写该文件。

### `AgentTrace[T]` —— 结果对象

每次 `.run()` 调用都会返回一个这样的对象：

```
身份标识：
  uuid         — 唯一运行 ID（来自 Claude 首条消息，或 OpenCode 的 session_id）
  session_id   — 会话标识符
  model        — 例如 "claude-sonnet-4-6"（来自 Claude）或来自 options（OpenCode）
  tools        — 工具列表（来自 Claude 首条消息或 options）

指标：
  duration_ms    — 运行耗时（OpenCode 为 0 —— 不上报）
  total_cost_usd — API 成本
  num_turns      — agent 进行了多少次工具调用（OpenCode 为 1 —— 不上报）
  usage          — token 计数字典

结果：
  result          — 原始文本响应
  is_error        — 若 agent 出错或结构化输出解析失败则为 True

结构化输出：
  output: T | None        — 解析后的 Pydantic 模型（例如带 final_answer 的 AgentResponse）
  parse_error: str | None — 解析失败的原因（例如 "ValidationError: ..."）
  raw_structured_output   — Pydantic 校验前的原始字典

调试：
  messages        — 来自 SDK 的完整消息列表（用于调试）
```

`summarize()` 方法生成一个文本版本供 proposer agent 阅读。成功时包含完整 trace。失败时截断为头部 + 尾部（默认各 60k 字符），以避免撑爆 proposer 的上下文窗口。

### `Agent[T]` —— 包装器

```python
agent = Agent(options=my_factory, response_model=AgentResponse)
trace = await agent.run("1940 年美国的国防开支是多少？")
```

**`__init__(options, response_model)`** —— 存储 options 提供者，以及用于校验输出的 Pydantic 模型。

**`_get_options()`** —— 若 `options` 是可调用对象，则调用它；否则直接返回。这正是工厂模式生效的原理。

**`_execute_query(query)`** —— SDK 分发点：

```python
if is_claude_sdk():
    from . import _claude_executor
    return await _claude_executor.execute_query(options, query)
else:
    from . import _opencode_executor
    return await _opencode_executor.execute_query(options, query)
```

当你添加 Goose/OpenHands 时，就在这里增加 `elif` 分支。

**`_run_with_retry(query)`** —— 为 `_execute_query` 包裹上韧性机制：

```
尝试 1：以 20 分钟超时运行查询
  → 成功？返回 messages
  → 超时？等待 30s，再试
  → 异常？等待 30s，再试

尝试 2：同上，但失败后等待 60s

尝试 3：同上，但失败后等待 120s

全部失败？抛出最后一个错误
```

**`run(query) → AgentTrace[T]`** —— 主入口：

```python
messages = await self._run_with_retry(query)    # 获取原始 SDK 消息

if is_claude_sdk():
    fields = _claude_executor.parse_response(messages, self.response_model)
else:
    fields = _opencode_executor.parse_response(messages, self.response_model, self._get_options)

return AgentTrace(**fields)    # 构造 SDK 无关的结果
```

---

## `_claude_executor.py` —— Claude SDK 专用逻辑

两个函数，都只被 `agent.py` 调用。

### `execute_query(options, query) → list[messages]`

1. 导入 `ClaudeSDKClient`（在函数内部，而非顶部）
2. 若 `options` 是字典，转换为 `ClaudeAgentOptions`（registry 路径的兜底）
3. 创建 `ClaudeSDKClient(options)` —— 这会启动一个 Claude Code 进程
4. `client.query(query)` —— 发送问题
5. `client.receive_response()` —— 随着消息流式返回，异步迭代它们
6. 将所有消息作为列表返回：`[SystemMessage, AssistantMessage, ..., ResultMessage]`

### `parse_response(messages, response_model) → dict`

从 Claude 的消息格式中提取 AgentTrace 字段：

```
messages[0]（SystemMessage）：
  .data["uuid"]   → uuid
  .data["model"]  → model
  .data["tools"]  → tools

messages[-1]（ResultMessage）：
  .session_id          → session_id
  .duration_ms         → duration_ms
  .total_cost_usd      → total_cost_usd
  .num_turns           → num_turns
  .usage               → usage
  .result              → result
  .is_error            → is_error
  .structured_output   → raw_structured_output → 校验 → output
```

结构化输出校验：
- 若 `structured_output` 非 None → 尝试 `response_model.model_validate(it)`
  - 成功 → `output = 解析后的 AgentResponse`
  - 失败 → `parse_error = "ValidationError: ..."`
- 若 `structured_output` 为 None → `parse_error = "No structured output returned (context limit likely exceeded)"`

返回一个字典，`Agent.run()` 会将其解包为 `AgentTrace(**fields)`。

---

## `_opencode_executor.py` —— OpenCode SDK 专用逻辑

比 Claude 更复杂，因为它要管理一个服务器的生命周期。

### 服务器管理

OpenCode 作为一个**本地 HTTP 服务器**运行（不同于 Claude 每次查询启动一个进程）。

**`_SERVER_PORTS: dict[str, int]`** —— 模块级字典，追踪每个项目使用的端口：
```python
{"/Users/me/project-a": 54321, "/Users/me/project-b": 54322}
```

**`_find_free_port()`** —— 向操作系统请求一个随机可用端口：
```python
sock.bind(("127.0.0.1", 0))  # 端口 0 = "给我任意一个空闲端口"
return sock.getsockname()[1]  # 例如 54321
```

**`_server_matches_project(client, expected_cwd)`** —— 询问一个运行中的服务器"你在哪个目录？"：
```python
app_info = await client.app.get()
# 检查 app_info.path.cwd 是否等于 expected_cwd
```
若服务器不可达或服务的是另一个项目，则返回 False。

**`_ensure_server(options)`** —— 主服务器生命周期函数：
```
1. 从 options 获取 requested_cwd
2. 查找该项目的端口（默认 4096）
3. 在该端口创建 client
4. 询问服务器："你服务的是我的项目吗？"
   是 → 返回 client（复用现有服务器）
   否 →
     a. 找一个空闲端口
     b. 启动：opencode serve --port {port} --hostname 127.0.0.1
     c. 等待 2 秒完成启动
     d. 创建新 client
     e. 返回 client
```

### `execute_query(options, query) → list[message]`

1. 校验 options 是字典（OpenCode 要求如此）
2. `_ensure_server(options)` —— 获取一个已连接的 client
3. `client.session.create()` —— 创建一个新会话
4. `client.session.chat(...)` —— 发送查询，携带：
   - `model_id` / `provider_id` —— 使用哪个 LLM
   - `parts` —— 查询文本
   - `system` —— system prompt
   - `mode` —— "build"（默认）
   - `tools` —— 可用工具
   - `extra_body` —— 结构化输出格式（若已配置）
5. 返回 `[message]` —— 单条消息包装为列表，以与 Claude 保持一致

### `parse_response(messages, response_model, get_options) → dict`

从 OpenCode 的消息格式中提取 AgentTrace 字段：

```
message.info：
  .get("structured_output")  → raw_structured_output（优先尝试）
  .get("structured")         → raw_structured_output（旧版本的兜底）
  .get("tokens", {})         → usage
  .get("cost", 0.0)          → total_cost_usd

message.parts：
  [{type: "text", text: "..."}]  → 拼接为 result_text

message.session_id  → uuid, session_id
```

**与 Claude 的关键区别**：OpenCode 的响应中不返回 `model` 或 `tools`。因此 `parse_response` 接收一个 `get_options` 可调用对象，从发送时的 options 中提取它们：

```python
options = get_options()
model_name = options.get("model_id", "unknown")
tools = list(options.get("tools", {}).keys())
```

此外：`duration_ms=0` 且 `num_turns=1` —— OpenCode 不上报这些。

---

## 一切如何串联运转

```
配置工厂调用 build_options(system=..., schema=..., tools=...)
    │
    ▼
build_options() 检查 get_sdk()
    │
    ├── "claude" → build_claudecode_options() → ClaudeAgentOptions
    └── "opencode" → build_opencode_options() → dict
    │
    ▼
Agent(options=result, response_model=AgentResponse)
    │
    ▼
agent.run("国防开支是多少？")
    │
    ▼
agent._run_with_retry(query)    ← 3 次尝试，20 分钟超时，指数退避
    │
    ▼
agent._execute_query(query)     ← 检查 is_claude_sdk()
    │
    ├── Claude: _claude_executor.execute_query(options, query)
    │     └── ClaudeSDKClient(options).query(query).receive_response()
    │     └── 返回 [SystemMessage, ..., ResultMessage]
    │
    └── OpenCode: _opencode_executor.execute_query(options, query)
          └── _ensure_server(options) → client
          └── client.session.create() → session
          └── client.session.chat(session.id, ...) → message
          └── 返回 [AssistantMessage]
    │
    ▼
agent.run() 解析响应      ← 再次检查 is_claude_sdk()
    │
    ├── Claude: _claude_executor.parse_response(messages, response_model)
    │     └── 从 SystemMessage + ResultMessage 提取
    │
    └── OpenCode: _opencode_executor.parse_response(messages, response_model, get_options)
          └── 从 AssistantMessage.info + .parts 提取
    │
    ▼
AgentTrace(**fields)              ← SDK 无关的结果
    │
    ▼
返回给调用方（循环、评估等）
```

---

## 添加一个新的 Harness（例如 Goose）

1. **创建 `_goose_executor.py`**，包含：
   - `execute_query(options, query) → list[Any]`
   - `parse_response(messages, response_model, ...) → dict`

2. **在 `options_utils.py` 中添加 `build_goose_options()`**

3. **在 `sdk_config.py` 中添加 "goose"**：扩展 `SDKType` 和 `set_sdk` 校验

4. **在 `options_utils.py:build_options()` 中添加分支**：
   ```python
   if sdk == "goose":
       return build_goose_options(...)
   ```

5. **在 `agent.py` 中添加分支**：
   ```python
   # 在 _execute_query 中：
   elif is_goose_sdk():
       from . import _goose_executor
       return await _goose_executor.execute_query(options, query)

   # 在 run 中：
   elif is_goose_sdk():
       from . import _goose_executor
       fields = _goose_executor.parse_response(messages, self.response_model, ...)
   ```

6. **无需改动 agent 配置、循环、评估、缓存、CLI 或 API。**
