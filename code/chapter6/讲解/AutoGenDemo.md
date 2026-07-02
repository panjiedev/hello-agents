# autogen_software_team.py 剥洋葱式讲解

> 源文件：`code/chapter6/AutoGenDemo/autogen_software_team.py`（194 行）
> 一句话概括：**用 AutoGen 组建一个四人"软件开发团队"（产品经理 → 工程师 → 代码审查员 → 用户代理），在一场轮流发言的群聊里协作完成"比特币价格显示应用"的开发任务。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

CAMEL 把协作限定在两人对话，这个文件把人数扩到四个，用的是 AutoGen 的核心隐喻——**群聊（GroupChat）**。整个程序的骨架：

1. 造一个模型客户端（第 20-26 行）；
2. 用四个 `create_xxx` 工厂函数造四个智能体，每个智能体 = 名字 + 一段系统提示词（第 28-114 行）；
3. 把四人拉进一个 `RoundRobinGroupChat` 群聊，扔进去一个任务，等他们聊出结果（第 116-172 行）。

AutoGen 的世界观是"**会话即程序**"（conversation as computation）：不写工作流代码，把分工写进各自的系统提示词，让消息在群里流动，任务就自然被接力完成。

---

## 🧅 第二层：智能体工厂——提示词即岗位说明书（第 28-101 行）

三个 `AssistantAgent` 的创建代码结构完全相同，以产品经理为例：

```python
return AssistantAgent(
    name="ProductManager",
    model_client=model_client,
    system_message=system_message,
)
```

（第 47-51 行）三个参数就是 AutoGen 定义 Agent 的全部要素：**名字、脑子（共享同一个 model_client）、人设**。三个智能体的差异 100% 来自 `system_message`——同一个模型，换一份"岗位说明书"就变成另一个专家。这延续了 chapter4 的结论：智能体的能力不在代码里，在提示词里。

更关键的是每份提示词的**最后一句话**：

- 产品经理：*"分析完成后说'请工程师开始实现'"*（第 45 行）
- 工程师：*"完成后说'请代码审查员检查'"*（第 70 行）
- 审查员：*"完成后说'代码审查完成，请用户代理测试'"*（第 95 行）

这些"交接暗号"在 `RoundRobinGroupChat` 里其实**不控制发言顺序**（顺序由轮询机制硬性决定，见第三层），但它们让每个智能体明确知道自己在流水线中的位置，产出"可被下一棒接住"的内容——提示词在这里承担的是**协作语义**而非控制流。

---

## 🧅 第三层：`UserProxyAgent`——人类的座位（第 103-114 行）

```python
return UserProxyAgent(
    name="UserProxy",
    description="""用户代理，负责以下职责：
    ...
    完成测试后请回复 TERMINATE。""",
)
```

第四个成员不是 `AssistantAgent` 而是 `UserProxyAgent`——AutoGen 为"人类参与"预留的接口。它没有 `model_client`，轮到它发言时会**暂停群聊，等真人在终端输入**。这是 AutoGen 区别于其他框架的招牌设计：human-in-the-loop 不是外挂，而是群聊里平等的一个席位。`description` 里的"完成测试后请回复 TERMINATE"是写给屏幕前那个真人看的操作说明。

---

## 🧅 第四层：组队与终止条件（第 133-145 行）

```python
termination = TextMentionTermination("TERMINATE")

team_chat = RoundRobinGroupChat(
    participants=[
        product_manager,
        engineer,
        code_reviewer,
        user_proxy
    ],
    termination_condition=termination,
    max_turns=20,
)
```

两个机制值得剥开：

- **`RoundRobinGroupChat`（轮询群聊）**：发言顺序 = `participants` 列表顺序，循环往复。产品经理 → 工程师 → 审查员 → 用户代理 → 再回产品经理……**列表的排列顺序就是软件开发的流水线顺序**，这是本文件里最隐蔽也最重要的一行"编排代码"。AutoGen 还提供 `SelectorGroupChat`（由 LLM 动态挑选下一个发言者），本例选轮询是为了流程可预测。
- **双保险终止**：`TextMentionTermination("TERMINATE")` 监听群聊中任何消息里出现暗号立即散会（暗号由用户代理/真人发出）；`max_turns=20` 兜底防止聊不完。又是那条铁律——**提示词（和人）定协议，框架做解析**。

---

## 🧅 洋葱心：一行启动整场协作（第 167 行）

```python
result = await Console(team_chat.run_stream(task=task))
```

这一行是整个 Demo 的引擎点火：

- `team_chat.run_stream(task=task)` 把任务（第 148-160 行那段比特币应用需求）作为第一条群聊消息广播出去，返回一个**异步消息流**——群聊每产生一条消息就吐出一条；
- `Console(...)` 订阅这个流，实时把每条消息打印到终端，实现"围观整场会议"的效果；
- `await` 直到终止条件触发才返回最终结果。

对比 CAMEL 需要自己写 `while` 循环逐轮 `step()`，AutoGen 把循环也收进了框架——用户代码里**看不到任何循环**，只有一次"点火"。整个文件用 `asyncio.run()` 驱动（第 178 行），因为 AutoGen 0.4+ 全面转向了异步事件驱动架构。

---

## 🎯 剥完之后：它在本章的位置

AutoGen 展示的是"**组织行为学**"式的多智能体设计：不画流程图，先设计一个组织（谁参加、按什么规则发言、开完会的标志是什么），然后把任务扔进会议室。

| 框架关键问题 | AutoGen 的回答 |
|---|---|
| Agent 怎么定义？ | `AssistantAgent(name, model_client, system_message)`，人设即岗位 |
| 多 Agent 怎么通信？ | 群聊广播，人人可见全部历史 |
| 流程谁控制？ | GroupChat 的发言者选择策略（本例为轮询）+ 文本终止条件 |
| 人类在哪？ | `UserProxyAgent`，群聊中的平等席位 |

它比 CAMEL 灵活（任意人数、可插入真人），但控制流仍是"软"的——你无法保证工程师一定在审查之后返工。想要**硬编码的、可回溯的确定性流程**，就要看下一篇 LangGraph 的"图"。
