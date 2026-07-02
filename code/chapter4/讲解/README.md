# Chapter 4 代码剥洋葱讲解

对 `code/chapter4/` 下五个代码文件的逐层拆解。每篇文档都采用"剥洋葱"结构：从最外层的**这个文件在干什么**开始，一层层剥到最核心的**那几行关键代码**，最后回到全局，说明它在智能体范式版图中的位置。

## 推荐阅读顺序

先看两块基石（脑与手），再依次看三种经典智能体范式：

| 顺序 | 文档 | 源文件 | 主题 | 一句话 |
|---|---|---|---|---|
| 1 | [llm_client.md](llm_client.md) | `llm_client.py` | LLM 客户端封装 | 智能体的"脑"：可复用的 `think()` 接口 + 流式响应 |
| 2 | [tools.md](tools.md) | `tools.py` | 搜索工具与工具注册表 | 智能体的"手"：真实联网搜索 + 可插拔工具箱 |
| 3 | [ReAct.md](ReAct.md) | `ReAct.py` | ReAct 范式 | 边想边做：Thought → Action → Observation 循环 |
| 4 | [Plan_and_solve.md](Plan_and_solve.md) | `Plan_and_solve.py` | Plan-and-Solve 范式 | 先谋后动：Planner 拆解 + Executor 逐步执行 |
| 5 | [Reflection.md](Reflection.md) | `Reflection.py` | Reflection 范式 | 做完再审：写代码 → 自我评审 → 迭代优化 |

## 一条主线

本章讲的是**如何组织 LLM 的思考**。三种范式各占一角，又能互补组合：

> **ReAct**（侦探破案：边查边想，灵活但缺全局观）
> **Plan-and-Solve**（工程师画图纸：先规划后执行，有全局观但计划僵化）
> **Reflection**(作家改稿：迭代精进，质量高但 token 开销成倍)

贯穿五个文件的一条工程铁律：**提示词定协议，代码做解析，两头严丝合缝**——ReAct 的 `Finish[...]`、Plan-and-Solve 的 ` ```python ` 代码块、Reflection 的"无需改进"暗号，都是同一思想的三次落地。

> 前置阅读：chapter3 的讲解（`code/chapter3/讲解/`）解释了 LLM 本身的原理；本章把 LLM 当作现成的"大脑"，关注怎么给它装上手脚和工作流。
