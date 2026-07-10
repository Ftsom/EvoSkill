# 评估层（Evaluation Layer）

评估文件夹分为 3 层：
1. **Runner（运行器）** —— 如何在大量题目上运行 agent（`evaluate.py`、`eval_full.py`）
2. **Reward function（奖励函数）** —— 如何判定答案是否正确（`reward.py`）
3. **任务专用评分器** —— 针对特定基准测试的自定义评分

---

## 第 1 层：在题目上运行 Agent

### `evaluate.py` —— 由自改进循环使用

```python
evaluate_agent_parallel(agent, items, max_concurrent=2, cache=None) → list[EvalResult]
```

- 接收 `(question, ground_truth)` 对，并行运行 agent
- 每题 **17 分钟超时**
- **缓存支持**：运行前检查 `cache.get(question)`，运行后通过 `cache.set()` 存储
- **优雅失败**：超时/异常 → `trace = None`（0 分，而非崩溃）
- 使用 `asyncio.Semaphore` 限制并发

结果类型：
```python
@dataclass
class EvalResult:
    question: str
    ground_truth: str
    trace: AgentTrace | None    # None = 失败/超时
```

### `eval_full.py` —— 由 `evoskill eval` 和独立脚本使用

```python
evaluate_full(agent, items, output_path, max_concurrent=5, resume=True) → list[IndexedEvalResult]
```

重型版本，额外具备以下特性：
- **items 携带索引**：追踪对应哪一行 CSV
- **增量落盘**：每题结束后追加写入 pickle 文件（崩溃安全）
- **断点续跑模式**：重启时跳过已成功的索引，只重跑失败项

| Runner | 使用者 | 缓存？ | 续跑？ | 落盘？ |
|--------|---------|--------|---------|----------------|
| `evaluate_agent_parallel` | 循环（`runner.py`） | 是（RunCache） | 否 | 否 |
| `evaluate_full` | `evoskill eval`、脚本 | 否 | 是（pickle） | 是（增量） |

---

## 第 2 层：奖励函数（`reward.py`）

入口：
```python
score_answer(ground_truth, predicted, tolerance=0.0) → 1.0 或 0.0
```

### 决策树

```
ground truth 中是否包含数字？
│
├── 是：prediction 中是否包含数字？
│   │
│   ├── 是：ground truth 中有几个数字？
│   │   │
│   │   ├── 多个（例如 "10 and 20"）：
│   │   │   所有 ground truth 数字都必须出现在 prediction 中
│   │   │   每个都在容差范围内 且 文本必须重叠
│   │   │
│   │   └── 单个（例如 "4.2 million"）：
│   │       1. 提取基础数字 + 单位（4.2, "million"）
│   │       2. 从 prediction 中过滤掉年份型数字（1900-2100）
│   │          除非 ground truth 本身就是年份
│   │       3. 找出容差范围内最接近的匹配
│   │       4. 若 GT 含文本，则检查文本重叠
│   │          （"March 1977" → 月份也必须匹配）
│   │
│   └── 否：→ 0.0
│
└── 否：纯文本比较
    1. 转小写、去除引号
    2. 移除括号内缩写，如 "(OASI)"
    3. 检查 GT 是否为 prediction 的子串
    4. 检查是否完全匹配
```

### 辅助函数

**`extract_numbers_with_context(text)`** —— 提取数字及其前后各 ±20 个字符的上下文：
```
"Revenue was 4.2 billion" → [(4.2, "revenue was 4.2 billion", False, False)]
```

**`detect_unit_in_context(context)`** —— 查找单位词：
```
"4.2 billion" → ("billion", 1e9)
"543 million" → ("million", 1e6)
"42"          → (None, 1.0)
```

**`normalize_number_with_units(number, context)`** —— 返回基础数字 + 单位。不做乘法："543 million" → `(543, "million")`，而非 `543000000`。

**`is_likely_year(num)`** —— num 是否在 1900-2100 之间且为整数？用于过滤偶然出现的年份引用。

**`has_significant_text(text)`** —— 文本中是否有数字/单位之外的词？
```
"March 1977"   → True（"march" 是有意义的）
"543 million"  → False（仅有数字 + 单位）
```

**`check_text_overlap(gt, pred)`** —— 对于像 "March 1977" 这样的混合答案，检查文本部分是否匹配：
```
GT="March 1977", Pred="March 1977" → True
GT="March 1977", Pred="April 1977" → False
GT="March 1977", Pred="1977"       → False
```

---

## 第 3 层：任务专用评分器

### `sealqa_scorer.py` —— LLM 充当评委

```python
score_sealqa(question, ground_truth, predicted) → 0.0 或 1.0
```

使用 GPT-5-mini（通过 OpenRouter）为答案评分：
1. 用 CORRECT/INCORRECT/NOT_ATTEMPTED 的示例填充评分模板
2. 发送给 LLM，得到 "A"、"B" 或 "C"
3. "A" = 1.0，其他一律 = 0.0

为什么不用字符串匹配？因为 SEAL-QA 的答案在语义上很复杂 —— "San Francisco" 应当匹配 "San Francisco, California"。

### `dabstep_scorer.py` —— 数值 + 字符串匹配

```python
question_scorer(input1, input2) → True/False
```

决策流程：
1. 含逗号的数字 → 提取数值，用 `math.isclose(rel_tol=1e-4)` 比较
2. 列表（包含 `;` 或 `,`）→ 拆分、排序、逐对比较
3. 普通数字 → 舍入后做数值比较
4. 字符串 → 完全匹配、词集包含检查，或 `SequenceMatcher` 相似度 > 0.95

### `livecodebench_scorer.py` —— 在 Docker 中执行代码

```python
score_livecodebench(question, ground_truth, predicted) → 0.0 或 1.0
```

步骤：
1. 从响应中提取 Python 代码（正则：` ```python ... ``` `）
2. 从 ground_truth 解析测试用例（JSON：`[{input, output}, ...]`）
3. 在 Docker 沙箱中运行代码（`llm_sandbox.SandboxSession`，每个测试 5 秒超时）
4. 将 stdout 与期望输出比对（完全匹配）
5. Pass@1：所有测试都必须通过 → 1.0，否则 0.0

---

## 循环如何使用评分

### 多容差评分（默认）

循环的默认评分器在 5 个容差级别上调用 `score_answer`，并取加权平均：

```
容差：[0%, 1%, 2.5%, 5%, 10%]
权重：[1.0, 0.83, 0.67, 0.5, 0.33]  （计算方式为 1/(1+20*tol)）
```

完全匹配 ≈ 1.0，接近的答案 ≈ 0.3-0.5，错误答案 = 0.0。

### 两种评分场景

1. **失败检测**（训练样本）：`score < 0.8` → 判为失败，送给 proposer
2. **程序评估**（验证集）：所有题目的平均分，用于与 frontier 比较
