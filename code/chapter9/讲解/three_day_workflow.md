# 06_three_day_workflow 剥洋葱式讲解

> 源文件：`code/chapter9/06_three_day_workflow.py`（291 行）
> 一句话概括：**用一个"三天 + 一周后"的剧本，验证 CodebaseMaintainer 的长程能力——尤其是最难的那一环：会话结束、进程退出之后，智能体凭什么还"记得"这个项目。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

前面所有示例都在单次运行里打转。但"长程智能体"的含金量恰恰在时间轴上：今天探索、明天分析、后天规划、下周复盘——中间隔着无数次进程退出。

这个文件是一场**排练**：它不定义任何新类，只是按剧本驱动 `CodebaseMaintainer` 走完一个真实的多天维护流程，并用两段额外演示（跨会话连贯性、三工具协同）把本章的核心命题验证给你看。舞台就是 `codebase/` 那个埋了 13 处 TODO 的示例代码库。

开头第 15-16 行还有个实用细节：

```python
os.environ['EMBED_MODEL_TYPE'] = 'tfidf'
os.environ['EMBED_MODEL_NAME'] = ''  # 重要：必须清空
```

用 TF-IDF 当嵌入模型——不用下载模型、不用 API key，示例的可运行性优先于检索质量。

---

## 🧅 第二层：三天剧本——同一个方法，三种意图（第 39-128 行）

三天各是一个函数，但仔细看，它们对 maintainer 做的事本质相同——**只给高层目标，不给操作步骤**：

| 天 | 调用 | Agent 被期待自主做的事 |
|---|---|---|
| 第一天 | `maintainer.explore()` | 自己决定用 find/ls/cat 摸清结构 |
| 第二天 | `maintainer.analyze()` | 自己决定 grep TODO、统计行数、给出改进建议 |
| 第三天 | `maintainer.plan_next_steps()` | 自己回顾历史笔记、制定计划 |

每天的第二个动作都是一句自然语言追问（如第 61 行"请查看 data_processor.py 文件，分析其代码设计"）。代码里反复出现的 `💡 提示：Agent 会自主决定……` 不是给用户看的客套，而是本文件的主题句：**编排层退化成了"出题人"，解题过程完全下放给 Agent**。

第三天有个值得注意的细节（第 120-124 行）：用户在提示词里明说"请使用 NoteTool 创建一个 task_state 类型的笔记来记录这个计划"——当某个动作必须发生时，与其赌 Agent 自觉，不如在指令里点名。**自主性和确定性之间，用一句话就能调节滑杆。**

---

## 🧅 第三层：一周后——复盘看的是账本，不是回忆（第 131-148 行）

```python
summary = maintainer.note_tool.run({"action": "summary"})
report = maintainer.generate_report()
```

"一周后检查进度"没有再问 LLM 一句话，而是直接查两本账：笔记摘要（各类型笔记的数量分布）和会话报告（执行了多少命令、创建了多少笔记）。这里有个朴素的道理：**进度不该靠模型"回忆"，而该靠外部记录直接读取**——凡是可以确定性获取的信息，就不要浪费一次 LLM 调用，更不要冒幻觉的风险。

---

## 🧅 第四层：三工具协同——一句话触发一条链（第 204-241 行）

```python
response = maintainer.run(
    "请分析代码库中的所有 TODO 项，并将发现记录到笔记中。"
    "然后告诉我应该优先实现哪些功能。"
)
```

一个请求，三个动词："分析"（TerminalTool 去 grep）、"记录"（NoteTool 去 create）、"告诉我"（LLM 综合作答）。没有任何代码规定这个顺序——`FunctionCallAgent` 在 `max_tool_iterations=30` 的额度内自己串起这条链。最后打印的统计（tool_calls / commands_executed / notes_created）就是验证：工具确实被自主调用了。

---

## 🧅 洋葱心：两次实例化之间的空气（第 159-192 行）

```python
# 第一次会话
maintainer_1 = CodebaseMaintainer(project_name="demo_codebase", ...)
maintainer_1.create_note(title="代码质量问题",
    content="发现多处 TODO 注释需要实现...", note_type="blocker", ...)

# （会话结束，maintainer_1 的一切内存状态作废）

# 第二次会话 (新的会话ID, 但笔记被保留)
maintainer_2 = CodebaseMaintainer(project_name="demo_codebase", ...)  # 同一个项目名!
response = maintainer_2.run("我们之前发现了什么代码质量问题？现在应该优先处理哪些？")
```

整章最关键的证据就藏在这两次实例化**之间**：`maintainer_1` 的对话历史、Agent 状态、session_id 全部随对象消亡；`maintainer_2` 是彻头彻尾的新对象、新会话。它凭什么答得上"我们之前发现了什么"？

链条是：相同的 `project_name` → 相同的笔记目录 `./demo_codebase_notes` → `run()` 第一步 `_retrieve_relevant_notes` 捞出那条 blocker 笔记（blocker 类型还享受 0.9 的最高相关性）→ 转成 ContextPacket → 装进新会话第一轮的上下文。

**跨会话延续从来不是魔法，而是"状态落盘 + 身份约定 + 启动时回注"三步的工程组合。** 会话是易失内存，笔记目录是持久磁盘，`project_name` 就是那把重新找到磁盘的钥匙。

---

## 🎯 剥完之后：它在本章的位置

这个文件是本章的**验收测试**，三段演示分别验收三个命题：

| 演示 | 验收的命题 | 底层支撑 |
|---|---|---|
| 三天剧本 | Agent 能只凭高层目标自主干活 | FunctionCallAgent + 模式提示 |
| 跨会话连贯性 | 会话死了，项目记忆还在 | NoteTool 落盘 + 启动回注 |
| 三工具协同 | 一句话能触发多工具链式协作 | ToolRegistry + 自主决策 |

读完它再回看章标题"上下文工程"就通了：**所谓长程智能体，就是把"记忆"从模型的上下文窗口里搬出来，存进文件系统，再在每一轮需要时精准地搬回去——搬运的艺术，就是上下文工程。**
