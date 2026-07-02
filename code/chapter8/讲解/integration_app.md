# 集成与应用（07、08、11）剥洋葱式讲解

> 源文件：
> - `code/chapter8/07_RAGTool_Intelligent_QA.py`（572 行）
> - `code/chapter8/08_Agent_Tool_Integration.py`（467 行）
> - `code/chapter8/11_Q&A_Assistant.py`（448 行）
>
> 一句话概括：**从零件到整机——07 剥开 RAG 问答的"检索 → 上下文构建 → 生成 → 引用"内部流程，08 演示两个工具如何注册进 Agent 并协同作战，11 把一切组装成带 Web 界面的 PDF 学习助手成品。**

---

## 🧅 第一层（最外层）：这三个文件在干什么？

前面的篇章把 MemoryTool 和 RAGTool 各自打磨好了，但零件不等于产品。这三个文件回答三个递进的问题：

| 文件 | 问题 | 层次 |
|---|---|---|
| 07 | RAG 的 `ask` 里面到底发生了什么？答案质量怎么衡量？ | 单工具深潜 |
| 08 | 两个工具怎么挂到 Agent 上？怎么配合干活？ | 工具编排 |
| 11 | 用户真正能用的产品长什么样？ | 完整应用 |

---

## 🧅 第二层：ask 的内脏——从问题到带引用的答案（07）

07 号文件先建了一个横跨 AI 基础、编程实践、系统设计三个领域的知识库（第 29-180 行），然后把 `ask` 动作的内部流程摊开。

**上下文构建**是 RAG 生成质量的命门（第 257-262 行列出五步）：

```
1. 🔍 检索相关文档片段     ← 向量检索（可开 MQE/HyDE）
2. 📊 按相关性排序
3. 🧹 清理和格式化内容
4. ✂️ 智能截断保持完整性   ← max_chars 控制，不能把句子切一半
5. 🔗 添加引用信息
```

对应的调用把所有旋钮拧到位（第 281-287 行）：

```python
qa_result = self.rag_tool.run({"action":"ask",
                                "question":complex_question,
                                "limit":4,                       # 取几个片段
                                "enable_advanced_search":True,   # MQE/HyDE 加持
                                "include_citations":True,        # 带上"参考来源"
                                "max_chars":1500})               # 上下文预算
```

`max_chars` 背后是 RAG 的核心张力：**上下文塞得越多，信息越全但噪音越多、token 越贵**——预算管理是必修课。

07 最有工程味的一段是**质量评分函数**（第 372-393 行）——用纯代码给 LLM 的输出打分：

```python
content_score  = (命中的期望要点数 / 期望要点总数) * 4.0   # 内容完整性 40%
length_score   = min(len(answer) / 500, 1.0) * 3.0         # 答案长度   30%
citation_score = 2.0 if "参考来源" in answer else 0.0       # 引用完整性 20%
speed_score    = max(0, 1.0 - (response_time - 1.0) / 5.0)  # 响应速度   10%
```

指标粗糙（关键词命中、字符数），但思想珍贵：**LLM 输出必须可度量，否则一切优化都是玄学**。第 502-518 行的引用系统对比（同一问题带引用/不带引用各答一遍）则点出可追溯性的价值——企业场景里，"答案从哪来"往往比答案本身更重要。

---

## 🧅 第三层：挂载——工具进入 Agent 的三步（08 第 27-55 行）

08 号文件用十几行代码完成了本章与第四章的会师：

```python
# 1. 造工具
self.memory_tool = MemoryTool(user_id="agent_integration_user",
                              memory_types=["working", "episodic", "semantic", "perceptual"])
self.rag_tool = RAGTool(knowledge_base_path="./agent_integration_kb",
                        rag_namespace="agent_demo")

# 2. 造 Agent（第四章的老朋友）
self.llm = HelloAgentsLLM()
self.agent = SimpleAgent(name="智能学习助手", llm=self.llm, ...)

# 3. 注册挂载
self.tool_registry = ToolRegistry()
self.tool_registry.register_tool(self.memory_tool)
self.tool_registry.register_tool(self.rag_tool)
self.agent.tool_registry = self.tool_registry
```

第四章的 ToolRegistry 当时装的是搜索工具，现在装的是记忆和知识库——**注册表模式的威力就在于此：Agent 一行不改，能力随插随用**。第 94-99 行的 `list_tools()` / `get_tool()` 展示了配套的发现机制：Agent 可以在运行时询问"我手上有什么工具"。

两个工具能被同一个注册表收编，靠的是统一接口（第 135-138、160-163 行做了并排对比）：`memory.run({"action":...})` 和 `rag.run({"action":...})` 调用形状完全一致——第 01 篇埋下的"字典协议"伏笔在这里兑现。

---

## 🧅 第四层：协同——1 + 1 > 2 的四个场景（08 第 165-337 行）

单个工具是能力，两个工具的配合才是智能。第 197-241 行的"学习新知识"场景是最典型的双写模式：

```python
# 同一次学习，两个系统各记各的
rag_result = self.rag_tool.run({"action":"add_text",
                                 "text":learning_content,          # 观察者模式的知识本身
                                 "document_id":"observer_pattern"})
memory_result = self.memory_tool.run({"action":"add",
                                       "content":"学习了观察者设计模式的定义、结构和应用场景",
                                       "memory_type":"episodic", ...})  # "我学过它"这件事
```

**RAG 存"是什么"，Memory 存"我经历了什么"**——回顾时反过来：Memory 检索唤起"我学过观察者模式"，RAG 补充"它的内容是……"。

第 269-337 行把协同升格为**编排**：完成"制定机器学习学习计划"这个复杂任务，工具链是 RAG 问知识结构 → Memory 记计划 → Memory 检索过往经验 → RAG 生成最终建议——四步交替调用两个工具，每一步的输出喂给下一步。这正是 Agent 面对复杂任务时该有的行为模式。

---

## 🧅 第五层：成品——PDF 学习助手（11）

11 号文件不再是演示脚本，而是一个真正能跑的产品：`PDFLearningAssistant` 类 + Gradio Web 界面。

**初始化**（第 26-48 行）：一人一份记忆（`user_id` 隔离），一人一个知识库命名空间（`rag_namespace=f"pdf_{user_id}"`），外加 `session_id` 标记本次会话——多用户数据互不串门。

**加载文档**（第 66-87 行）：`add_document` 处理 PDF（chunk_size=1000 / overlap=200），成功后顺手往情景记忆里记一笔"加载了文档《xxx》"——从第一个动作起，系统就在写自己的日记。

**Web 层的意图路由**（第 284-291 行）藏着一个朴素而有效的设计：

```python
if any(keyword in message for keyword in ["之前", "学过", "回顾", "历史", "记得"]):
    response = assistant.recall(message)   # → Memory：查自己的经历
else:
    response = assistant.ask(message)      # → RAG：查文档的知识
```

一行关键词判断，把"问知识"和"问经历"分流到两套系统——这就是双系统架构在用户界面上的投影。

此外 `add_note`（第 153-160 行）把用户笔记写入**语义记忆**（`memory_type="semantic"`），`generate_report`（第 207-228 行）汇总 Memory 摘要 + RAG 统计生成学习报告——四种记忆里的三种都在产品里找到了岗位。

---

## 🧅 洋葱心：ask 方法的"一次提问，三笔记录"（11 第 113-142 行）

整个应用最浓缩的一段，是 `PDFLearningAssistant.ask` 的三连击：

```python
# ① 问题本身 → 工作记忆（短期，会话内有效）
self.memory_tool.run({"action":"add", "content":f"提问: {question}",
                       "memory_type":"working", "importance":0.6, ...})

# ② 真正干活：RAG 检索 + 生成答案
answer = self.rag_tool.run({"action":"ask", "question":question, "limit":5,
                             "enable_advanced_search":use_advanced_search,
                             "enable_mqe":use_advanced_search,
                             "enable_hyde":use_advanced_search})

# ③ 这次学习交互 → 情景记忆（长期，跨会话留存）
self.memory_tool.run({"action":"add", "content":f"关于'{question}'的学习",
                       "memory_type":"episodic", "importance":0.7, ...})
```

用户只看到一问一答，系统内部却完成了**知识检索 + 双层记忆写入**。正因为每次提问都留痕，"我之前学过什么？"才有据可查——本章所有概念（四种记忆的分工、RAG 链路、MQE/HyDE 开关、工具协同）在这三次 `run` 里全部到齐。**这十几行就是整个第八章的合影。**

---

## 🎯 剥完之后：它在本章的位置

这三个文件是本章的收官：07 把 RAG 问答的黑盒拆成白盒，08 把两个工具接上 Agent 的"神经系统"，11 把一切装进用户可以点击的界面。

回望全章的一条暗线：**所有能力都收敛于同一个 `run({"action": ...})` 协议**——正是这个统一契约，让 MemoryTool 和 RAGTool 能被 ToolRegistry 一视同仁地管理，能被 Agent 无差别地调用，能在应用层自由编排。第四章说"智能体的能力在提示词里"，本章补上了另一半：**智能体的记性，在工具里**。
