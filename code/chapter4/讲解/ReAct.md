# ReAct.py 剥洋葱式讲解

> 源文件：`code/chapter4/ReAct.py`（99 行）
> 一句话概括：**实现 ReAct（Reasoning + Acting，推理与行动交织）范式——让 LLM 一边想、一边用工具查、一边根据查到的结果修正想法，是现代智能体的奠基性设计。**

第一章的 `FirstAgentTest.py` 已经手搓过一个 ReAct 雏形；本文件是它的"工程化重构版"：提示词模板化、工具可插拔、解析逻辑封装成类。对比着读收获最大。

---

## 🧅 第一层（最外层）：这个文件在干什么？

问 LLM"华为最新的手机是哪一款？"，它只能凭训练数据猜（多半过时）。ReAct 的解法是让模型进入一个循环：

```
Thought（我需要搜索最新信息）
  → Action: Search[华为最新手机]         ← 模型"说"出想做的动作
  → Observation: [1] 华为 Mate 80 …      ← 程序真的去搜，把结果喂回去
  → Thought（我知道了，可以回答了）
  → Action: Finish[华为最新的手机是…]
```

运行本文件（`__main__`，第 92-99 行），就是装配好脑（`HelloAgentsLLM`）和手（`ToolExecutor` + Search 工具），对这个问题跑一遍上述循环。

**关键认知：LLM 自始至终只会生成文本。**"调用工具"是幻觉——模型只是按格式说出 `Search[...]`，真正执行的是外面这层 Python 代码。ReAct 智能体 = LLM + 一个"当真了"的解释器。

---

## 🧅 第二层：提示词模板——智能体的"操作系统"（第 6-24 行）

```
可用工具如下：
{tools}

请严格按照以下格式进行回应：
Thought: 你的思考过程...
Action: 你决定采取的行动，必须是以下格式之一：
- `{{tool_name}}[{{tool_input}}]`：调用一个可用工具。
- `Finish[最终答案]`：当你认为已经获得最终答案时。

Question: {question}
History: {history}
```

三个插槽（slot）各司其职：

| 插槽 | 填什么 | 作用 |
|---|---|---|
| `{tools}` | `ToolExecutor.getAvailableTools()` 生成的工具清单 | 告诉模型**能做什么** |
| `{question}` | 用户的原始问题 | 告诉模型**目标是什么** |
| `{history}` | 累积的 Action/Observation 轨迹 | 告诉模型**已经发生了什么** |

细节：模板里的 `{{tool_name}}` 用了**双花括号**——`str.format()` 中双花括号是转义，输出为字面量 `{tool_name}`，因为这里是在向模型**展示格式**，而不是要 Python 填值。

这份模板就是智能体的全部"程序"：约束输出格式（方便解析）+ 注入工具与历史（提供上下文）。改这段文本，就是在"重新编程"这个智能体。

---

## 🧅 第三层：`run` 主循环——智能体的心跳（第 33-73 行）

剥开 `run` 方法，每一次循环迭代是五拍：

```python
while current_step < self.max_steps:
    # ① 组装提示词：工具清单 + 问题 + 历史轨迹
    prompt = REACT_PROMPT_TEMPLATE.format(tools=..., question=..., history=...)
    # ② 请模型思考
    response_text = self.llm_client.think(messages=...)
    # ③ 解析出 Thought 和 Action
    thought, action = self._parse_output(response_text)
    # ④ 分派：Finish → 返回答案；工具调用 → 真的执行
    if action.startswith("Finish"): return final_answer
    observation = tool_function(tool_input)
    # ⑤ 把 Action 和 Observation 写回历史，进入下一圈
    self.history.append(f"Action: {action}")
    self.history.append(f"Observation: {observation}")
```

三个设计决策值得剥出来：

- **`max_steps=5` 是安全阀**。模型可能陷入"反复搜索同一个东西"的死循环，步数上限保证智能体一定会停机——所有生产级 Agent 框架都有这个参数。
- **历史是唯一的记忆**。LLM 本身无状态，每一步都是把"到目前为止的完整剧本"重新喂给它一遍。智能体的"连贯性"完全靠 `{history}` 插槽维持。
- **错误也是 Observation**（第 62、66 行）。Action 格式无效？喂回 `"Observation: 无效的Action格式，请检查。"`；工具不存在？喂回 `"错误：未找到名为 'xxx' 的工具。"`。**不崩溃、不修正，而是把错误告诉模型让它自己纠错**——这是 ReAct 最优雅的地方：负反馈也是推理的燃料。

---

## 🧅 洋葱心：三个解析函数——正则是 LLM 和程序的"边境海关"（第 75-90 行）

模型输出的是自由文本，程序需要的是结构化指令，中间隔着三个正则：

### ① `_parse_output`：切出 Thought 和 Action

```python
thought_match = re.search(r"Thought:\s*(.*?)(?=\nAction:|$)", text, re.DOTALL)
action_match  = re.search(r"Action:\s*(.*?)$", text, re.DOTALL)
```

剥开 Thought 的正则：`(.*?)` 是**非贪婪**匹配，`(?=\nAction:|$)` 是**先行断言**——"一直取到遇见 `\nAction:` 或文本结尾为止，但不把它们吃进匹配结果"。`re.DOTALL` 让 `.` 能匹配换行，因为思考过程可能写好几行。

### ② `_parse_action`：拆出工具名和参数

```python
re.match(r"(\w+)\[(.*)\]", action_text, re.DOTALL)
# "Search[华为最新手机]" → ("Search", "华为最新手机")
```

### ③ `_parse_action_input`：从 `Finish[...]` 中取出最终答案

**为什么说这是洋葱心？** 因为这条"文本 → 结构"的边境线是所有 LLM 应用最脆弱的地方：模型少写一个冒号、多编一种格式，解析就失败。本代码的对策是三层防线——提示词里"请严格按照以下格式"（事前约束）、解析失败后把错误喂回历史（事中纠错）、`max_steps` 兜底（事后止损）。后来的 Function Calling / JSON mode 本质上就是把这道正则海关下沉到了模型服务内部，但原理与此完全相同。

---

## 🎯 剥完之后：ReAct 的位置与局限

ReAct（Yao et al., 2022）确立了智能体的最小完备循环：**思考 → 行动 → 观察 → 再思考**。第一章的手搓版、本章的工程版、乃至你正在用的各种 Copilot，骨架都是它。

它的局限也清晰：**走一步看一步，没有全局规划**——复杂任务容易绕远路；每一步都携带全部历史，token 消耗随步数线性膨胀。这两个问题，正是同目录下两个文件的主题：

- `Plan_and_solve.py`：先规划全局，再逐步执行；
- `Reflection.py`：干完之后回头审视，迭代精进。
