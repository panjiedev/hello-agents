# codebase_maintainer 剥洋葱式讲解

> 源文件：`code/chapter9/codebase_maintainer.py`（477 行）
> 一句话概括：**本章的集大成者——把 ContextBuilder（装配）、NoteTool（外部记忆）、TerminalTool（即时检索）、MemoryTool（工作记忆）拧成一个能跨会话维护代码库的长程智能体，且不预定义工作流，一切由 Agent 自主决策。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

前面的示例每次只演示一两个组件。`CodebaseMaintainer` 回答的是终极问题：**这些零件怎么组装成一个真正能干活的长程智能体？**

它的任务场景：维护一个代码库（探索结构 → 分析质量 → 规划重构），任务跨越多天多次会话。用法只有三个动词：

```python
maintainer = CodebaseMaintainer(project_name="...", codebase_path="...", llm=...)
maintainer.explore()          # 探索
maintainer.analyze()          # 分析
maintainer.plan_next_steps()  # 规划
```

文件开头的注释点明了它与旧式设计的分野（第 10 行）："使用 Agentic 方式，让 agent 自主决定使用哪些工具"——代码不再写死"先 ls 再 cat 再记笔记"，而是把工具和目标都交给 Agent，让它自己编排。

---

## 🧅 第二层：__init__——四件套的组装图（第 37-101 行）

```python
self.memory_tool = MemoryTool(user_id=project_name, memory_types=["working"])
self.note_tool = NoteTool(workspace=f"./{project_name}_notes")
self.terminal_tool = TerminalTool(workspace=codebase_path, timeout=60)

self.context_builder = ContextBuilder(
    memory_tool=self.memory_tool,
    rag_tool=None,  # 本案例不使用 RAG
    config=ContextConfig(max_tokens=4000, reserve_ratio=0.15,
                         min_relevance=0.2, enable_compression=True)
)
```

三个身份绑定值得注意：

- **MemoryTool 绑 `project_name`**，且只开 `working` 记忆——免去 Qdrant 向量库依赖，示例开箱即跑；
- **NoteTool 的 workspace 也由 `project_name` 派生**——这是跨会话延续的伏笔：只要项目名相同，新会话就打开同一个笔记目录；
- **TerminalTool 的 workspace 绑 `codebase_path`**——Agent 的手只能伸进被维护的代码库。

随后三个工具注册进 `ToolRegistry`，交给 `FunctionCallAgent`（第 71-84 行，`max_tool_iterations=30`，允许 Agent 一口气连环调用 30 次工具）。而 `session_id` 用时间戳生成（第 45 行）——**每次会话 ID 都不同，但笔记目录相同**，"会话易失、笔记永存"的对比由此埋下。

---

## 🧅 第三层：run()——每一轮的五步节拍（第 103-151 行）

```python
# 第一步: 检索相关笔记（为 agent 提供上下文）
relevant_notes = self._retrieve_relevant_notes(user_input)
note_packets = self._notes_to_packets(relevant_notes)

# 第二步: 构建优化的上下文
context = self.context_builder.build(
    user_query=user_input,
    conversation_history=self.conversation_history,
    system_instructions=self._build_system_instructions(mode),
    additional_packets=note_packets
)

# 第三步: 让 Agent 自主决策和使用工具
self.agent.system_prompt = context
response = self.agent.run(user_input)
```

之后第四步统计工具使用、第五步更新对话历史（保留最近 20 条，第 344-346 行）。

这里有一个精妙的**双层上下文架构**：

- **外层（推上下文）**：run() 主动把历史笔记、对话历史、模式提示装配好，塞进 Agent 的 system_prompt——模型还没开口，该记得的都已在眼前；
- **内层（拉上下文）**：Agent 在执行中若发现信息不够，自己调 TerminalTool 现场去拿，调 NoteTool 现场去记。

推是"喂给你该知道的"，拉是"你自己去查想知道的"——长程智能体的上下文供给必须两条腿走路。

---

## 🧅 第四层：差异化相关性与模式提示（第 254-332 行）

笔记转上下文包时，不同类型给不同分（第 262-267 行）：

```python
relevance_map = {
    "blocker": 0.9,      # 阻塞问题最优先
    "action": 0.8,       # 行动计划次之
    "task_state": 0.75,  # 任务状态
    "conclusion": 0.7    # 结论垫底
}
```

对比 04 里一刀切的 0.75，这里进化成了**按语义类型分层的优先级**：token 预算紧张时，"卡住的事"永远比"已得的结论"先进上下文——这是任务管理常识在评分函数里的直接编码。

`_build_system_instructions(mode)` 则演示了 Agentic 时代的"软引导"（第 293-332 行）：`explore` / `analyze` / `plan` 三种模式各附一段"建议策略"（如 analyze 模式提示"考虑用 grep 查找 TODO、FIXME"）。注意措辞全是"考虑""建议"——**引导方向而不写死步骤**，与 `explore()` / `analyze()` / `plan_next_steps()` 三个便捷方法（第 350-370 行）一一对应：便捷方法只是"高层目标 + 模式提示"的封装，具体怎么做仍由 Agent 决定。

顺带一提 `_track_tool_usage`（第 179-191 行）：靠在消息内容里找 "terminal" / "note" 等关键词做统计，是启发式的粗略计数——统计报表允许模糊，上下文构建不允许，两处代码的严谨度差异本身就是一课。

---

## 🧅 第五层：被维护的对象——codebase/ 子包

`codebase/` 是专门造出来"给 Agent 练手"的示例代码库，四个模块共约 350 行、埋了 13 处 TODO：

| 模块 | 内容 | 埋的坑 |
|---|---|---|
| `data_processor.py` | Pandas 数据清洗/转换/聚合 | 4 个 TODO（缺数据验证等）|
| `api_client.py` | API 客户端 | 3 个 TODO（错误处理薄弱）|
| `models.py` | 数据模型 | 4 个 TODO（缺字段验证）|
| `utils.py` | 工具函数 | 2 个 TODO（待优化）|

TODO 就是故意留的"病灶"：Agent 一条 `grep -rn 'TODO'` 就能列出问题清单，正好把探索→分析→规划的全流程跑通。

---

## 🧅 洋葱心：动态换脑的两行（第 137-140 行）

```python
self.agent.system_prompt = context
response = self.agent.run(user_input)
```

全文件最核心的动作：**每一轮都用 ContextBuilder 新装配的上下文，整体替换 Agent 的 system_prompt，然后才放它去自主行动。**

传统 Agent 的 system_prompt 是出厂即固定的"人设"；这里它变成了**每轮重写的动态简报**——[Role & Policies] 里有身份和工具说明，[Context] 里有历史对话和昨天的 blocker 笔记，[Task] 里有本轮目标。Agent 的"自主性"（自己选工具、自己定步骤）和"连贯性"（记得之前发生过什么）在这两行完成合流：

> 上下文工程负责让它**记得**，Function Calling 负责让它**能干**，两者的接缝就是这一次赋值。

---

## 🎯 剥完之后：它在本章的位置

`CodebaseMaintainer` 是本章所有概念的落地闭环：

| 组件 | 在这里扮演 | 对应前置示例 |
|---|---|---|
| ContextBuilder | 每轮装配"简报" | 01 / 02 |
| NoteTool | 跨会话的项目档案 | 03 / 04 |
| TerminalTool | 伸进代码库的手 | 05 |
| MemoryTool | 会话内工作记忆 | （第八章）|
| FunctionCallAgent | 自主决策的大脑 | （第七章）|

它本身又是 `06_three_day_workflow.py` 的主角——那份脚本会驱动这里的 explore/analyze/plan 跑完完整的三天，并用两次实例化证明"会话可以死，项目记忆不死"。
