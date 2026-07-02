# 01/02_context_builder 剥洋葱式讲解

> 源文件：`code/chapter9/01_context_builder_basic.py`（103 行）+ `code/chapter9/02_context_builder_with_agent.py`（102 行）
> 一句话概括：**ContextBuilder 把"该给模型看什么"从一次性的字符串拼接，升级为一条可配置的流水线——先独立演示（01），再装进 Agent 的每一轮对话里（02）。**

---

## 🧅 第一层（最外层）：这两个文件在干什么？

前几章的智能体都有一个隐藏假设：把系统提示词 + 全部对话历史一股脑塞给 LLM 就行。任务一长这个假设就崩了——上下文窗口有限，历史越堆越多，真正相关的信息反而被淹没。

上下文工程的第一课就是：**上下文不是"攒出来的"，是"构建出来的"**。每一轮对话前，都应该重新回答一个问题——在有限的 token 预算内，哪些信息最值得进入模型的视野？

- `01_context_builder_basic.py` 单独演示 `ContextBuilder`：喂进去对话历史 + 用户问题 + 系统指令，吐出来一个**分区结构化的上下文字符串**；
- `02_context_builder_with_agent.py` 把它接进 `SimpleAgent`，让每一轮 `run()` 都自动走一遍"构建→调用→更新历史"的循环。

---

## 🧅 第二层：ContextConfig——预算和门槛的四个旋钮（01 第 30-35 行）

```python
config = ContextConfig(
    max_tokens=3000,
    reserve_ratio=0.2,
    min_relevance=0,  # 最小相关性阈值，0代表所有历史信息会被保留
    enable_compression=True
)
```

四个参数各管一件事：

| 参数 | 管什么 | 直觉 |
|---|---|---|
| `max_tokens` | 总预算 | 上下文最多花多少 token |
| `reserve_ratio` | 预留比例 | 先给系统指令留出 20% 的"包厢"，不许别人挤占 |
| `min_relevance` | 入场门槛 | 相关性低于此值的信息直接淘汰（示例设 0，即全放行）|
| `enable_compression` | 兜底压缩 | 万一还是超了，最后再截断一次 |

这是典型的**预算式思维**：不是问"这条信息要不要"，而是问"在 3000 token 里它排第几"。

---

## 🧅 第三层：build()——一次调用，四步流水线（01 第 70-74 行）

```python
context_str = builder.build(
    user_query="如何优化Pandas的内存占用?",
    conversation_history=conversation_history,
    system_instructions="你是一位资深的Python数据工程顾问。..."
)
```

对调用方来说只是一个函数，内部（实现在 `hello_agents` 包里，详见教材 9.3 节）却是四步流水线：

1. **Collect（汇集）**——把系统指令、对话历史、记忆、RAG 结果统统封装成 `ContextPacket`（内容 + 时间戳 + token 数 + 相关性分数的统一四元组）；
2. **Select（选择）**——对每个包算 `相关性 × 权重 + 新近性 × 权重` 的综合分，按分数从高到低**贪心填充**，直到预算用尽（系统指令免评分、必保留）；
3. **Structure（结构化）**——把选中的包按类型装进五个分区；
4. **Compress（压缩）**——若仍超限，按分区逐段截断，保住结构完整性。

最终输出的字符串长这样：

```
[Role & Policies]   ← 系统指令：你是谁、守什么规矩
[Task]              ← 本轮用户问题
[Evidence]          ← RAG/知识库证据（本例未启用）
[Context]           ← 对话历史 + 记忆
[Output]            ← 输出要求
```

**这五个分区就是"上下文分区组装"的具体形态**：不同来源的信息各居其位，模型和人都能一眼看清"哪句话来自哪里"——可读、可调试、可扩展。

注意 01 中 `MemoryTool` / `RAGTool` 全部被注释掉了（第 25-26、54-66 行）——这是刻意的：先看清骨架（历史 + 指令也能构建），再按需插电（记忆、知识库都是可选插槽）。

---

## 🧅 第四层：构建结果怎么用？（01 第 85-89 行）

```python
messages = [
    {"role": "system", "content": context_str},
    {"role": "user", "content": "请回答"}
]
```

一个容易被忽视的设计决定：**整个构建好的上下文（包括 [Task] 里的用户问题）都塞进 system message**，user message 只剩一句"请回答"。这样做的好处是上下文的所有分区在同一个位置、按统一模板呈现——LLM 面对的是一份"简报"而不是散落各处的碎片。

---

## 🧅 第五层：装进 Agent——每轮重建（02 第 38-71 行）

02 的 `ContextAwareAgent` 继承 `SimpleAgent`，重写 `run()` 为固定四步：

```python
def run(self, user_input: str) -> str:
    # 1. 构建优化的上下文
    optimized_context = self.context_builder.build(
        user_query=user_input,
        conversation_history=self.conversation_history,
        system_instructions=self.system_prompt
    )
    # 2. 调 LLM
    # 3. 更新对话历史
    # 4. (可选)记入记忆系统
```

关键的范式转变藏在第 1 步：**上下文不是持久对象，而是每轮的临时产物**。历史（`conversation_history`）才是持久的原材料，每次 `run()` 都从原材料现做一份"当前最优上下文"。第二轮问"能给出具体的代码示例吗?"（第 95 行）时，第一轮的问答已经躺在历史里，会被重新评分、重新装配进 `[Context]` 分区——多轮连贯性由此而来。

---

## 🧅 洋葱心：build 的三个入参（02 第 42-46 行）

```python
optimized_context = self.context_builder.build(
    user_query=user_input,
    conversation_history=self.conversation_history,
    system_instructions=self.system_prompt
)
```

这三个参数恰好对应上下文的三种时间尺度：

- `system_instructions`——**永恒的**（角色与规则，轮轮不变）；
- `conversation_history`——**积累的**（越滚越长，需要筛选）;
- `user_query`——**即时的**（本轮焦点，同时也是给历史评分的"尺子"）。

ContextBuilder 的全部工作，就是让这三种时间尺度的信息在固定预算下达成平衡。后面的 NoteTool（跨天）和 TerminalTool（即时文件）不过是往这个平衡里再加两种信息源。

---

## 🎯 剥完之后：它在本章的位置

ContextBuilder 是本章的**地基**：

| 后续文件 | 怎么用它 |
|---|---|
| `04_note_tool_integration.py` | 把笔记转成 `ContextPacket`，经 `additional_packets` 注入分区 |
| `codebase_maintainer.py` | 每轮先 build 上下文，再交给会调工具的 Agent |
| `06_three_day_workflow.py` | 三天工作流的每一次交互底下都是它在装配 |

记住一条：**模型每一轮"看到的世界"是被设计出来的**。设计得好，4000 token 能顶别人 40000；设计得差，再大的窗口也只是更大的垃圾场。
