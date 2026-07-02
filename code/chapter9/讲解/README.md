# Chapter 9 代码剥洋葱讲解

对 `code/chapter9/` 下七个示例文件的逐层拆解。每篇文档都采用"剥洋葱"结构：从最外层的**这个文件在干什么**开始，一层层剥到最核心的**那几行关键代码**，最后回到全局，说明它在上下文工程版图中的位置。

## 推荐阅读顺序

先看装配层（上下文怎么被构建），再看两种上下文来源（外部记忆与即时检索），最后看它们如何合体成一个真正的长程智能体：

| 顺序 | 文档 | 源文件 | 主题 | 一句话 |
|---|---|---|---|---|
| 1 | [context_builder.md](context_builder.md) | `01_context_builder_basic.py`<br>`02_context_builder_with_agent.py` | 上下文构建器 | Collect→Select→Structure→Compress 四步流水线，在 token 预算内装配出五分区的结构化上下文 |
| 2 | [note_tool.md](note_tool.md) | `03_note_tool_operations.py`<br>`04_note_tool_integration.py` | 结构化笔记 | 笔记作为外部记忆：Markdown 落盘存状态，检索→转包→注入形成读写闭环 |
| 3 | [terminal_tool.md](terminal_tool.md) | `05_terminal_tool_examples.py` | 即时文件访问 | JIT 上下文：白名单 shell 命令按需探索文件系统，Unix 管道充当免费的上下文压缩器 |
| 4 | [codebase_maintainer.md](codebase_maintainer.md) | `codebase_maintainer.py`<br>（含 `codebase/` 示例代码库） | 综合实战 | 四件套拧成长程智能体：每轮重建的上下文整体替换 Agent 的 system_prompt，自主性与连贯性在此合流 |
| 5 | [three_day_workflow.md](three_day_workflow.md) | `06_three_day_workflow.py` | 三天工作流 | 验收测试：三天剧本验证自主决策，两次实例化验证"会话可以死，项目记忆不死" |

> 上手建议：`03` 和 `05` 不需要配置 LLM 就能直接运行，可以边跑边读。

## 一条主线

本章讲的是**如何管理 LLM 的视野**。上下文窗口有限且易失，三个组件各补一块短板：

> **ContextBuilder**（编辑部：每轮在预算内重排版面，决定模型看什么）
> **NoteTool**（工作日志：把精确状态写成文件，跨会话不遗忘）
> **TerminalTool**（现场勘查：不预载不索引，现用现查，管道先做压缩）

贯穿七个文件的一条工程主线：**模型每一轮"看到的世界"是被设计出来的**——信息按来源分区组装（[Role & Policies] / [Task] / [Evidence] / [Context] / [Output]），按"相关性 × 新近性"竞争预算，按类型（blocker > action > task_state > conclusion）划分优先级；而跨会话的延续从来不是魔法，只是"状态落盘 + 身份约定（project_name）+ 启动时回注"的三步组合。

> 前置阅读：chapter4 的讲解（`code/chapter4/讲解/`）解释了智能体范式（怎么思考），第七、八章提供了 Function Calling 与记忆系统（怎么动手、怎么记住）；本章解决的是长程任务的最后一块拼图——**怎么在有限的窗口里，始终看见对的东西**。
