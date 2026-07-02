# Chapter 15 代码剥洋葱讲解

对 `code/chapter15/Helloagents-AI-Town/` 综合项目（AI 小镇）的逐层拆解。每篇文档都采用"剥洋葱"结构：从最外层的**这个模块在干什么**开始，一层层剥到最核心的**那几行关键代码**，最后回到全局，说明它在整个项目中的位置。

本章是综合实战：一个 **Python 后端（FastAPI + HelloAgents）+ Godot 前端**的小镇模拟游戏——三个 NPC（张三、李四、王五）住在办公室里，有记忆、有好感度、有自己的日程；玩家走近按 E 就能聊天，聊过的话他们会记住，态度也会随相处改变。

## 推荐阅读顺序

先立骨架，再进灵魂，然后看关系与世界如何运转，最后走到玩家眼前：

| 顺序 | 文档 | 源文件 | 主题 | 一句话 |
|---|---|---|---|---|
| 1 | [backend_skeleton.md](backend_skeleton.md) | `backend/main.py` `config.py` `models.py` `logger.py` `view_logs.py` | 后端骨架 | 小镇的"市政厅"：薄路由 + Pydantic 契约 + 全程日志 |
| 2 | [town_agents.md](town_agents.md) | `backend/agents.py` | 智能体与记忆 | 本章灵魂：好感度 + 记忆检索 + 提示词拼装的六步对话流水线 |
| 3 | [relationship_and_state.md](relationship_and_state.md) | `backend/relationship_manager.py` `state_manager.py` `batch_generator.py` | 关系与状态 | LLM 当裁判判好感度；一次调用批量生成三个 NPC 的"日子" |
| 4 | [godot_frontend.md](godot_frontend.md) | `helloagents-ai-town/scripts/*.gd` | Godot 前端 | 小镇的"肉身"：移动、巡逻、对话框，靠 HTTP+JSON 与后端解耦 |

（`memory_data/` 是 NPC 记忆的 SQLite 落盘目录，`assets/` 与 `.tscn` 场景文件是美术资源，各 `*_GUIDE.md` 是运行指南，均不在剥解范围内。）

## 一条主线

本章讲的是**AI 小镇——生成式智能体的记忆、关系与自主行为**。三个机制各答一问：

> **记忆**（agents.py：对话落盘 SQLite，下次以 RAG 方式检索注入——NPC 记得你说过什么）
> **关系**（relationship_manager.py：裁判 LLM 判分 → 好感度积累 → 翻译成语气指令——NPC 对你日久生情或渐生嫌隙）
> **自主行为**（batch_generator.py + state_manager.py：30 秒心跳批量生成台词——没人搭理时 NPC 也过自己的日子）

贯穿全项目的两条工程主张：

1. **状态在 Agent 之外，按需织入提示词**——`SimpleAgent` 本身无状态，NPC 的"人格连续性"全靠运行时把记忆和好感度拼进上下文（town_agents.md 的洋葱心）；
2. **游戏引擎与 LLM 后端隔 HTTP 解耦，智能分级供给**——昂贵的语言走 `/chat` 即时生成，廉价的巡逻本地随机，群演台词批量生产降本 66%；换模型不动前端，换引擎不动后端（godot_frontend.md 的洋葱心）。

> 前置阅读：chapter4 的讲解（`code/chapter4/讲解/`）解释了智能体如何思考；本章把会思考的智能体放进一个持续运转的世界，让它拥有过去（记忆）、立场（关系）和生活（自主行为）——这正是 Generative Agents（斯坦福 AI 小镇）思想的一次小型工程复刻。
