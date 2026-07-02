# Chapter 6 代码剥洋葱讲解

对 `code/chapter6/` 下四个多智能体框架 Demo 的逐层拆解。每篇文档都采用"剥洋葱"结构：从最外层的**这个 Demo 在干什么**开始，一层层剥到最核心的**那几行关键代码**，最后回到全局，说明该框架在多智能体版图中的位置。

## 推荐阅读顺序

按"用户代码由少到多、控制权由框架到开发者"的顺序读，正好走完一条从"全托管"到"自己拼积木"的光谱：

| 顺序 | 文档 | 源文件 | 主题 | 一句话 |
|---|---|---|---|---|
| 1 | [CAMEL.md](CAMEL.md) | `CAMEL/DigitalBookWriting.py` | 角色扮演协作 | 报两个角色名，作家与心理学家一问一答合写电子书 |
| 2 | [AutoGenDemo.md](AutoGenDemo.md) | `AutoGenDemo/autogen_software_team.py` | 群聊式团队 | 四个岗位提示词 + 轮询群聊 = 一条软件开发流水线 |
| 3 | [Langgraph.md](Langgraph.md) | `Langgraph/Dialogue_System.py` | 图式工作流 | 把"理解→搜索→回答"画成状态图，确定性执行 + 检查点记忆 |
| 4 | [AgentScopeDemo.md](AgentScopeDemo.md) | `AgentScopeDemo/*.py`（5 个文件） | 消息基础设施 | MsgHub 分组广播 + pipeline 调度 + 结构化输出，拼出一局三国狼人杀 |

## 一条主线

本章讲的是**四大多智能体框架的设计哲学对比**。同一个问题——"多个 LLM 如何协作"——四家给出了四种世界观：

> **CAMEL**（相声搭档：框架内置角色扮演协议，对话即协作，简单但拓扑焊死在两人）
> **AutoGen**（开会：把智能体拉进群聊，靠发言规则和交接暗号推进，灵活但流程是"软"的）
> **LangGraph**（画流程图：节点 + 边 + 共享状态，流程硬编码进图里，确定可控但少了涌现）
> **AgentScope**（搭乐高：通信、调度、格式约束三种正交积木，最自由也最需要自己动手）

对照读四篇会发现三组反复出现的取舍：

1. **谁定义 Agent**——CAMEL 里框架生成提示词，其余三家都是"名字 + 系统提示词 + 模型"三件套，能力全在提示词里（chapter4 的老结论）；
2. **谁控制流程**——CAMEL/AutoGen 交给对话涌现（配文本暗号 `CAMEL_TASK_DONE` / `TERMINATE` 终止），LangGraph/AgentScope 交给开发者的显式编排；
3. **输出怎么保真**——从 CAMEL/AutoGen 的自由文本，到 LangGraph 的"提示词约定 + split 解析"，再到 AgentScope 的 Pydantic `structured_model`，协议一步步从提示词升格到类型系统——**"提示词定协议，代码做解析"这条铁律，在框架时代进化成了"Schema 定协议，框架做解析"。**

> 前置阅读：chapter4 的讲解（`code/chapter4/讲解/`）手写了单智能体的三种范式；本章把"组织 LLM 思考"升级为"组织一群 LLM 协作"，框架们做的正是把 chapter4 里手写的循环、解析、兜底代码沉淀为基础设施。
