# BFCL 评估三部曲剥洋葱式讲解

> 源文件：`code/chapter12/02_bfcl_quick_start.py`（50 行）、`03_bfcl_custom_evaluation.py`（62 行）、`04_run_bfcl_evaluation.py`（294 行）
> 一句话概括：**用 BFCL（Berkeley Function Calling Leaderboard）量化"智能体会不会正确调用函数"——三个文件是同一件事的三档深度：一键评估、拆开评估、对齐官方评估。**

---

## 🧅 第一层（最外层）：这三个文件在干什么？

BFCL 是 UC Berkeley 推出的函数调用基准：给智能体一个问题和一堆函数定义，看它能否生成**正确的函数调用**（函数名对、参数名对、参数值对）。判分不靠人眼，靠 **AST 匹配**算法。

三个文件是三级火箭：

| 文件 | 深度 | 适合谁 |
|---|---|---|
| `02_bfcl_quick_start.py` | `bfcl_tool.run()` 一行搞定 | 只想拿个分数 |
| `03_bfcl_custom_evaluation.py` | 拆成 Dataset + Evaluator | 想看每道题错在哪 |
| `04_run_bfcl_evaluation.py` | 五步流水线，接入 BFCL **官方**评估工具 | 想让分数能上榜、能横向对比 |

---

## 🧅 第二层：一键评估——评估工具也是"工具"（02 第 17-28 行）

```python
bfcl_tool = BFCLEvaluationTool()
results = bfcl_tool.run(
    agent=agent,
    category="simple_python",  # 评估类别
    max_samples=5              # 评估样本数（0表示全部）
)
print(f"准确率: {results['overall_accuracy']:.2%}")
```

一个有趣的设计：`BFCLEvaluationTool` 来自 `hello_agents.tools`——**评估器和搜索工具住在同一个包里**。在 HelloAgents 的世界观里，"给智能体打分"和"帮智能体搜索"一样，都是可插拔的工具。`max_samples=5` 是全章通用的省钱阀门：先用 5 个样本验证流程通、格式对，再放开跑全量（400 样本约 4-8 元）。

---

## 🧅 第三层：拆开看——Dataset 与 Evaluator 分家（03 第 18-36 行）

```python
dataset = BFCLDataset(
    bfcl_data_dir="./temp_gorilla/berkeley-function-call-leaderboard/bfcl_eval/data",
    category="simple_python"
)
evaluator = BFCLEvaluator(dataset=dataset, category=category)
results = evaluator.evaluate(agent=agent, max_samples=5)
```

02 里的一键工具在这里被拆成两个正交的组件：**Dataset 管"考什么"**（从克隆下来的 gorilla 仓库读官方题库），**Evaluator 管"怎么判"**。拆开的回报在第 46-52 行：

```python
for detail in results['detailed_results']:
    print(f"  预测: {detail['predicted']}")
    print(f"  正确答案: {detail['expected']}")
    print(f"  结果: {'✅ 正确' if detail['success'] else '❌ 错误'}")
```

一键评估只给总分，底层组件给你**逐题的 predicted vs expected 对照**——错误分析（是函数名选错了，还是参数值格式不对）只能在这一层做。

---

## 🧅 第四层：让 LLM 变成"纯函数调用器"（04 第 35-54、87-92 行）

04 是最完整的一份，先看它给智能体动的手术：

```python
FUNCTION_CALLING_SYSTEM_PROMPT = """你是一个专业的函数调用助手。
...
1. 必须是纯JSON格式，不要添加任何解释文字
2. 使用JSON数组格式：[{"name": "函数名", "arguments": {"参数名": "参数值"}}]
...
- 只输出JSON，不要添加"好的"、"我来帮你"等额外文字
"""

agent = SimpleAgent(
    name=model_name,
    llm=llm,
    system_prompt=FUNCTION_CALLING_SYSTEM_PROMPT,
    enable_tool_calling=False      # ← 关键
)
```

两个点值得剥开：

1. **`enable_tool_calling=False`**——BFCL 测的不是"真的执行了函数"，而是"能否**说出**正确的函数调用"。所以要关掉框架的工具执行，让 LLM 的裸输出（一段 JSON）直接进判分器。
2. **提示词里的每一句禁令都对应一种判分失败**："不要添加解释文字"是因为多一个"好的"，JSON 解析就炸；"参数名必须与函数定义完全一致"是因为 AST 匹配对函数名和参数名是**精确匹配**。又是那条铁律：提示词定协议，判分器做解析，两头必须严丝合缝。

---

## 🧅 第五层：五步流水线——接轨官方（04 第 249-288 行）

`main()` 把评估串成五步：

```
run_evaluation → export_bfcl_format → copy_to_bfcl_result_dir
             → run_bfcl_official_eval → show_results
```

前两步是 HelloAgents 自己评（自研判分），后三步把结果**喂给 BFCL 官方命令行工具重判一遍**（第 177-182 行）：

```python
cmd = ["bfcl", "evaluate", "--model", model_name,
       "--test-category", category, "--partial-eval"]
```

为什么要评两遍？因为**自研判分器说 100% 没有公信力，官方判分器说 100% 才能和排行榜上的模型对话**。为此要迁就官方工具的三个怪癖：结果文件必须叫 `BFCL_v4_{category}_result.json` 并放进 `result/<模型名>/` 目录（第 149-157 行）；模型名必须是 BFCL 认识的名字（默认 `Qwen/Qwen3-8B`，第 254 行），且路径里的 `/` 要替换成 `_`（第 148 行）；`--partial-eval` 允许只评部分样本。最后 `show_results` 从官方产出的 `score/` 目录里读回 CSV 和 JSON 评分（第 222-246 行）——`template_output/` 目录里躺着的就是这套流水线的真实产物。

---

## 🧅 洋葱心：AST 匹配——为什么不用字符串比对？

判分的核心算法不在本目录（在 hello-agents 框架的 `evaluation/bfcl/ast_matcher.py` 里），但它是三个文件共同的地基，必须剥到：

$$\text{AST\_Match}(P, G) = 1 \iff \text{AST}(P) \equiv \text{AST}(G)$$

把预测调用和标准答案各自解析成**抽象语法树**再比较，而不是比字符串。于是：

- `f(a=1, b=2)` ≡ `f(b=2, a=1)` —— 参数顺序无关（键值对当集合比）
- `f(x=2+3)` ≡ `f(x=5)` —— 等价表达式算对
- `f(s="hello")` ≡ `f(s='hello')` —— 引号风格无关
- 但 `get_weather` ≠ `get_temperature` —— 函数名必须精确匹配

准确率就是 AST 匹配成功的比例：$\text{Accuracy} = \frac{1}{N}\sum_i \text{AST\_Match}(P_i, G_i)$。**宽容语法上的表面差异，严格语义上的实质差异**——这一个设计决定了 BFCL 的分数既不冤枉好人（格式怪癖），也不放过坏人（调错函数）。

---

## 🎯 剥完之后：它在本章的位置

BFCL 回答的是第一个拷问：**智能体的"手"稳不稳**——工具调用是智能体区别于聊天机器人的根本能力，而 AST 匹配给了这个能力一个客观、可复现、每样本约一次 API 调用的廉价度量。它的局限也清晰：只考单步的"说出调用"，不考多步推理和真实执行。多步、真实世界的那半边，交给下一篇 GAIA（`gaia_evaluation.md`）。
