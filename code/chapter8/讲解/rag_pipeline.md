# RAG 管线（04、05、10）剥洋葱式讲解

> 源文件：
> - `code/chapter8/04_RAGTool_MarkItDown_Pipeline.py`（407 行）
> - `code/chapter8/05_RAGTool_Advanced_Search.py`（445 行）
> - `code/chapter8/10_RAG_Pipeline_Complete.py`（856 行）
>
> 一句话概括：**RAGTool 把"任意格式文档 → Markdown → 分块 → 嵌入 → 向量检索 → 生成答案"串成一条流水线，04 讲入库端，05 讲检索端的两把高级武器（MQE / HyDE），10 把全链路从头到尾走一遍。**

---

## 🧅 第一层（最外层）：这三个文件在干什么？

Memory 解决"记住自己的经历"，RAG（Retrieval-Augmented Generation，检索增强生成）解决另一个问题：**LLM 不知道你的私有文档**。让模型回答"我们公司的 API 怎么调"，靠预训练是没戏的，靠 RAG 是三步：把文档切碎存进向量库（入库）、按问题捞出最相关的碎片（检索）、把碎片塞进提示词让 LLM 作答（生成）。

三个文件各守一段：

| 文件 | 守哪一段 | 核心问题 |
|---|---|---|
| 04 | 入库 | 五花八门的格式怎么统一处理？怎么切块才不破坏语义？ |
| 05 | 检索 | 用户问得含糊时，怎么还能捞准？（MQE、HyDE） |
| 10 | 全链路 | 摄取 → 分块 → 检索 → 问答 → 性能优化，端到端串讲 |

和 MemoryTool 一样，RAGTool 也是字典协议一个入口：

```python
rag_tool.run({"action": "add_document" / "add_text" / "search" / "ask", ...})
```

---

## 🧅 第二层：入库端——Any 格式的"归一化漏斗"（04）

04 号文件先造了四个格式各异的文档：Markdown、HTML、JSON、CSV（第 31-118 行），然后全部喂给同一个动作（第 158-159 行）：

```python
result = self.rag_tool.run({"action":"add_document",
                             "file_path":file_path})
```

不管什么格式进来，内部第一站都是 **MarkItDown**——微软开源的"万物转 Markdown"库。为什么偏偏选 Markdown 做中间格式？因为它**天生带结构还足够干净**：`#`/`##`/`###` 保留了文档的层次骨架，又没有 HTML 那种标签噪音。归一化之后，后续的分块、嵌入只需要面对一种格式。

第二站是**结构感知的分块**（第 250-254 行）：

```python
result = self.rag_tool.run({"action":"add_text",
                             "text":complex_markdown,
                             "document_id":"ai_tech_stack",
                             "chunk_size":800,        # 每块目标 token 数
                             "chunk_overlap":100})    # 相邻块重叠量
```

两个参数是 RAG 工程里的经典权衡：

- `chunk_size`：块太大，检索粒度粗、无关内容混进上下文；块太小，语义被腰斩。800 是"一个小节"的量级。
- `chunk_overlap`：相邻块重叠 100 token，防止关键句恰好被切在边界上——**宁可存两遍，不可断一句**。

而"基于 Markdown 结构分块"意味着切割优先发生在标题边界上，一个 `### 监督学习` 小节尽量完整地落在同一块里——这正是 04 用嵌套四级标题的"人工智能技术栈"长文（第 186-247 行）来测试的原因。

第三站是**面向嵌入的预处理**（第 287-317 行）：入库前把 `**加粗**`、`[链接](url)`、代码块围栏这些格式记号剥掉，只留语义文本。嵌入模型对 `**重要的**` 里的星号毫无兴趣，留着只会稀释向量质量。

最后第 341-344 行的 `batch_add_texts` 展示批量入库——生产环境里文档是成百上千地来的。

---

## 🧅 第三层：检索端——当用户问得含糊时（05）

基础检索（第 198-201 行）就是"问题嵌入 → 向量库最近邻"，注意那个显式关掉高级功能的参数：

```python
result = self.rag_tool.run({"action":"search",
                             "query":query,
                             "limit":2,
                             "enable_advanced_search":False})
```

它工作得不错，直到用户问出"深度学习"这种两个词的模糊查询。此时两把高级武器登场，共同思想是：**查询和文档天然不长一个样，与其硬匹配，不如先用 LLM 把查询"整容"成更容易匹配的样子。**

**武器一：MQE（Multi-Query Expansion，多查询扩展）**——治"问得太窄"。用 LLM 把一个查询改写成多个语义等价的说法，并行检索再合并去重。开关就是那个布尔值（第 238-241 行）：

```python
mqe_result = self.rag_tool.run({"action":"search",
                                 "query":query,
                                 "limit":3,
                                 "enable_advanced_search":True})   # ← 只改了这一个值
```

"优化算法"会被扩展成"梯度下降方法"、"模型调参技巧"……哪个说法命中文档用哪个，召回率自然上去。

**武器二：HyDE（Hypothetical Document Embeddings，假设文档嵌入）**——治"问题和答案长得不像"。问题"Transformer 相比 RNN 有什么优势？"是疑问句，而文档里躺着的是陈述句。HyDE 让 LLM 先**凭空写一段假设性答案**（哪怕内容有幻觉也没关系），再拿这段假答案去做向量检索——因为**假答案和真文档在向量空间里是邻居，而问题不是**。05 里 HyDE 藏在 `ask` 动作内部（第 273-276 行）。

代价也在第 244-248 行摆明了：MQE/HyDE 每次检索都要多调一次 LLM，耗时是基础检索的数倍。**高级检索是拿延迟和 token 换质量**——什么时候开、什么时候关，是工程判断而非默认全开。

---

## 🧅 第四层：全链路走一遍（10）

10 号文件是集大成者，五段演示对应流水线五站：

**① 摄取（第 195-268 行）**：Markdown 教程、纯文本报告、JSON API 文档三种"文档人格"统一入库，且每份都带元数据透传：

```python
result = self.rag_tool.run({"action":"add_text",
                             "text":doc["content"],
                             "document_id":doc["document_id"],
                             **doc["metadata"]})     # title / author / date... 全部随块存储
```

元数据不是摆设——它让后续检索结果能回答"这段话出自哪份文档的哪一章"。

**② 分块（第 348-353 行）**：拿一篇四阶段的《人工智能发展史》长文测语义分块，再用"图灵测试是什么？"等定点问题验证：好的分块应该让每个问题恰好命中承载答案的那一块。

**③ 高级检索的"白盒演示"（第 460-501 行）**：这是 10 号文件最有教学价值的一段——它把 05 里封装在开关背后的 MQE 和 HyDE **手动展开**给你看：

```python
# MQE 手动版：一个查询变五个（第 460-478 行）
expanded_queries = ["机器学习模型性能优化方法", "提升ML模型准确率的技巧", ...]
for query in [base_query] + expanded_queries:
    results = self.rag_tool.run({"action":"search", "query":query, "limit":3})

# HyDE 手动版：拿假答案当查询（第 487-495 行）
hypothetical_answer = """深度学习是机器学习的一个子领域，它使用多层神经网络..."""
hyde_results = self.rag_tool.run({"action":"search",
                                   "query":hypothetical_answer,   # ← 查询不是问题，是假答案
                                   "limit":5})
```

看懂这两段，`enable_advanced_search=True` 那个黑盒开关就再无秘密。

**④ 智能问答（第 594-606 行）**：`ask` 动作把定义类、方法类、比较类、原理类、应用类五种问题统一处理，内部是"检索 → 上下文构建 → 生成"（这条链在第 07 篇里细剥）。

**⑤ 性能优化（第 764-777 行）**：同一查询跑两遍对比耗时，演示缓存命中——嵌入计算是 RAG 里最贵的重复劳动，缓存是第一性价比的优化。

---

## 🧅 洋葱心：三个 action 撑起一条管线

八百多行演示代码剥到最后，RAG 的全部骨架就是三次 `run`：

```python
rag_tool.run({"action":"add_text",  "text":doc, "chunk_size":800, "chunk_overlap":100})
                     # ↑ 入库：转 Markdown → 分块 → 嵌入 → 存向量库
rag_tool.run({"action":"search", "query":q, "enable_advanced_search":True})
                     # ↑ 检索：（MQE/HyDE 改写）→ 向量最近邻 → 相关块
rag_tool.run({"action":"ask", "question":q, "limit":5})
                     # ↑ 生成：检索 → 拼上下文 → LLM 作答（带引用）
```

三个动作对应 RAG 的三个不可省略的环节，而所有的工程巧思——MarkItDown 归一化、结构感知分块、重叠策略、查询改写、缓存——都是在这三根柱子上做的精装修。**记住这三行，就记住了 RAG。**

---

## 🎯 剥完之后：它在本章的位置

RAGTool 是本章"知识"这条线的主角：

| 相关文件 | 关系 |
|---|---|
| 07 智能问答 | 把 `ask` 动作的内部（上下文构建、引用系统、质量评估）拆开细看 |
| 08 Agent 集成 | RAGTool 和 MemoryTool 一起注册进 ToolRegistry，供 Agent 调度 |
| 11 问答助手 | 用 `add_document` 加载真实 PDF，做出一个可用的学习助手产品 |

一个耐人寻味的对照：RAG 的"分块-嵌入-检索"和 Memory 的"编码-存储-检索"其实是**同一套向量思想的两次落地**——只不过 RAG 面向外部文档（世界的知识），Memory 面向交互历史（自己的经历）。第 11 篇的问答助手正是把这两条线拧成一股绳。
