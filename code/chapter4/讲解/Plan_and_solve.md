# Plan_and_solve.py 剥洋葱式讲解

> 源文件：`code/chapter4/Plan_and_solve.py`（125 行）
> 一句话概括：**实现 Plan-and-Solve 范式——先让 LLM 把复杂问题拆成分步计划（Planner），再逐步执行（Executor），"三思而后行"的智能体。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

ReAct 是"走一步看一步"，遇到多步推理题容易中途迷路。Plan-and-Solve 换一种策略：

```
问题：水果店周一卖15个苹果，周二是周一的两倍，周三比周二少5个，三天共卖多少？

第一阶段 Planner：先拆解
  ["计算周二卖出的数量", "计算周三卖出的数量", "三天求和"]

第二阶段 Executor：按计划逐步算
  步骤1 → 30    步骤2 → 25    步骤3 → 70
```

运行本文件就是对上面这道数学题跑一遍完整流程。注意：**这个智能体没有任何外部工具**，两个阶段全靠 LLM 自身的推理——它展示的是"如何组织思考"，而不是"如何使用工具"。

---

## 🧅 第二层：`PlanAndSolveAgent`——只有装配，没有逻辑（第 101-115 行）

```python
class PlanAndSolveAgent:
    def __init__(self, llm_client):
        self.planner = Planner(self.llm_client)
        self.executor = Executor(self.llm_client)

    def run(self, question):
        plan = self.planner.plan(question)       # 第一阶段
        if not plan:
            print("...无法生成有效的行动计划。"); return
        final_answer = self.executor.execute(question, plan)  # 第二阶段
```

顶层类薄得几乎透明：造两个部件、先后调用、中间做一次防御检查（计划为空就终止，不带病执行）。**注意两个部件共享同一个 `llm_client`**——所谓"规划器"和"执行器"是两个不同的提示词人格，背后是同一个模型。分工不靠模型，靠提示词。

---

## 🧅 第三层：`Planner`——让模型输出"程序能吃的"计划（第 19-54 行）

规划提示词（第 19-30 行）的核心诉求只有一个：**输出必须是机器可解析的**。

```
你的输出必须是一个Python列表，其中每个元素都是一个描述子任务的字符串。
请严格按照以下格式输出你的计划，```python与```作为前后缀是必要的:
```python
["步骤1", "步骤2", "步骤3", ...]
```
```

为什么强制要求 ` ```python ` 代码块包裹？因为 LLM 天生爱寒暄（"好的，以下是我的计划：……"）。有了固定的前后缀标记，解析代码就能精准地把列表从闲聊中"抠"出来：

```python
plan_str = response_text.split("```python")[1].split("```")[0].strip()
plan = ast.literal_eval(plan_str)
```

`split` 两刀切出代码块内容——这是比正则更糙但更直观的抠取法。

### 这一层的宝石：`ast.literal_eval` 而不是 `eval`

模型返回的 `'["步骤1", "步骤2"]'` 是**字符串**，要变成真正的 Python 列表。`eval()` 也能做到，但它会执行任意代码——如果模型（或注入提示词的攻击者）返回 `__import__('os').system('rm -rf ~')`，`eval` 会照单全收。`ast.literal_eval` 只接受字面量（列表、字符串、数字等），遇到任何函数调用直接抛异常。**把 LLM 输出当作不可信输入来处理**，这是 LLM 工程的一条铁律，本文件用一个函数名教会你。

解析失败时返回空列表 `[]`，并打印原始响应方便排查——和 `llm_client` 的"返回 None 不抛异常"是同一种契约。

---

## 🧅 洋葱心：`Executor`——历史的滚雪球（第 77-99 行）

```python
def execute(self, question, plan):
    history = ""
    for i, step in enumerate(plan, 1):
        prompt = EXECUTOR_PROMPT_TEMPLATE.format(
            question=question, plan=plan,
            history=history if history else "无",
            current_step=step
        )
        response_text = self.llm_client.think(messages=...) or ""
        history += f"步骤 {i}: {step}\n结果: {response_text}\n\n"   # ← 雪球
        final_answer = response_text
    return final_answer
```

最核心的机制是 `history` 这个滚雪球：**每执行完一步，就把"步骤+结果"追加进历史；下一步的提示词里携带全部历史。** 这解决了步骤间的数据流问题——步骤 2 算"周三 = 周二 − 5"时，必须能看到步骤 1 得出的"周二 = 30"。LLM 无状态，所谓"记得上一步"，全靠提示词里这段不断变长的文本。

执行提示词（第 57-75 行）里还有两个容易剥漏的设计：

- **给全景，限焦点**：提示词同时给出原始问题、完整计划、历史结果，但明确要求"**请你专注于解决'当前步骤'**"。全景防止断章取义，限焦防止模型一口气把后面的步骤全做了（那样计划就形同虚设）。
- **"仅输出该步骤的最终答案，不要输出任何额外的解释"**：因为这一步的输出会原样进入下一步的提示词——输出越干净，历史雪球越不容易滚进噪声。

最后 `final_answer = response_text` 不断被覆盖，**默认最后一个步骤的输出就是最终答案**——这隐含了对 Planner 的一个要求：计划的最后一步必须是"汇总/得出最终答案"类的步骤。

---

## 🎯 剥完之后：与 ReAct 的对照

| | ReAct | Plan-and-Solve |
|---|---|---|
| 策略 | 走一步看一步（反应式） | 先谋后动（规划式） |
| 全局观 | 无，容易绕路 | 有，先看清全貌 |
| 灵活性 | 强：每步可根据 Observation 转向 | 弱：**计划一旦生成就不再修改** |
| 适合 | 探索性任务（信息不足，边查边想） | 结构清晰的多步推理（数学题、流程性任务） |

Plan-and-Solve 的软肋恰是它的卖点：计划是静态的，步骤 3 发现步骤 1 错了也只能硬着头皮走完。改进方向自然浮现——执行中发现问题就**回头修订计划**，甚至干脆对结果做系统性的"复盘"。后者正是下一个文件 `Reflection.py` 的主题。
