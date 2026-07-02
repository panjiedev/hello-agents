# HelloAgent_n8nCase.json 剥洋葱式讲解

> 源文件：`code/chapter5/HelloAgent_n8nCase.json`（350 行）
> 一句话概括：**一个 n8n"AI 邮件自动回复助手"——Gmail 每分钟轮询新邮件，AI Agent 联网搜答案、查工作时间知识库，自动写好回信发回去；核心看点是 n8n 独有的"能力插槽"式连线。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

设想一个场景：你在睡觉，有人发邮件问你问题。这个工作流让 AI 替你值夜班——它会：

1. 每分钟检查 Gmail 收件箱（第 4-30 行，`gmailTrigger` 节点）；
2. 让一个 AI Agent 阅读邮件，用 SerpAPI 联网搜索答案，同时查询"我的工作时间"知识库；
3. 如果当前是非工作时间，在回信开头声明"我会在下个工作日回复，以下是 AI 找到的初步答案"；
4. 自动 `Re:` 原主题回信给发件人（第 83-105 行，`gmail` 发送节点）。

整个 JSON 只有两个顶层关键块：`nodes`（11 个节点，第 3-226 行）和 `connections`（连线关系，第 228-339 行）。**读 n8n 导出文件的诀窍：先读 connections 再读 nodes**——连线的"类型"才是 n8n 的灵魂。

---

## 🧅 第二层：两条互不相连的流水线

11 个节点其实组成了**两张图**：

**知识写入线**（左侧，只需手动跑一次）：

```
Code in JavaScript1 ──main──→ Simple Vector Store1 (mode: insert)
                                   ↑ ai_document        ↑ ai_embedding
                          Default Data Loader    Embeddings Google Gemini
```

`Code in JavaScript1`（第 156-168 行）用一段 JS 硬编码了三条"个人知识"：

```javascript
{ "doc_id": "work-schedule-001",
  "content": "我的工作时间是周一至周五，上午9点到下午5点。时区是澳大利亚东部标准时间（AEST）。" },
{ "doc_id": "off-hours-policy-001", ... },
{ "doc_id": "auto-reply-instruction-001", ... }
```

这三条文本经 Gemini Embedding 向量化后，存入名为 `my-dailytime` 的内存向量库（第 106-123 行）。这就是一个**微型 RAG 的"建库"环节**。

**主线**（每封新邮件触发一次）：

```
Gmail Trigger ──main──→ AI Agent1 ──main──→ Send a message
```

主线上只有三个节点，但 AI Agent 底下"挂"着四个辅助节点——这就要看第三层了。

---

## 🧅 第三层：connections 里的"能力插槽"（第 228-339 行）

普通工作流平台只有一种边：数据边。n8n 有**六种**。看 `connections` 块里每条边的 `type`：

```json
"Gmail":                    { "main":             [[→ AI Agent1]] },
"Google Gemini Chat Model": { "ai_languageModel": [[→ AI Agent1]] },
"Simple Memory":            { "ai_memory":        [[→ AI Agent1]] },
"SerpAPI":                  { "ai_tool":          [[→ AI Agent1]] },
"Simple Vector Store2":     { "ai_tool":          [[→ AI Agent1]] },
"Embeddings Google Gemini1":{ "ai_embedding":     [[→ Simple Vector Store2]] }
```

- `main` 边：**数据流**，邮件内容从这里流入流出；
- `ai_languageModel` / `ai_memory` / `ai_tool` / `ai_embedding` / `ai_document` 边：**能力流**，它们不传数据，而是把"大脑、记忆、手、眼"插到 Agent 身上。

对照 chapter4：我们手写 ReAct 时，LLM 客户端、工具注册表都是构造函数参数——n8n 把"依赖注入"画成了不同颜色的连线。**AI Agent1 是插座，周围节点是插头。**

三个插头各有巧思：

- **Simple Memory**（第 50-63 行）：`sessionKey: ={{ $('Gmail').item.json.threadId }}`——用邮件线程 ID 当会话键，同一封邮件的往来共享记忆，不同线程互不串台；
- **Simple Vector Store2**（第 169-188 行）：`mode: retrieve-as-tool`——把上面建好的 `my-dailytime` 向量库**包装成一个工具**暴露给 Agent，`toolDescription` 字段就是写给 LLM 看的"工具说明书"（"当需要判断当前是否为工作时间……必须使用此工具"）。RAG 不再是固定流程，而是 Agent 想查才查；
- **SerpAPI**（第 64-82 行）：现成的联网搜索工具，等价于 chapter4 `tools.py` 里手写的那个搜索函数。

---

## 🧅 第四层：AI Agent 节点——提示词藏在这里（第 208-225 行）

`@n8n/n8n-nodes-langchain.agent` 节点（n8n 底层用的是 LangChain）有两段提示词：

`text` 字段是**用户消息模板**，用 n8n 表达式把邮件字段填进去：

```
- 当前时间: {{ new Date().toLocaleString('en-AU', { timeZone: 'Australia/Sydney', ... }) }}
- 发件人: {{ $json.From }}
- 主题: {{ $json.Subject }}
- 邮件正文: {{ $json.snippet }}
```

注意 `{{ }}` 里可以跑真正的 JavaScript（`new Date()...`）——**当前时间不是问模型的，是运行时算出来注入的**，这比让 LLM 自己"猜"时间可靠得多。

`options.systemMessage` 是**系统提示词**，结构非常完整：角色和目标 → 可用工具 → 执行步骤 → 规则和限制。其中最关键的两段：

```
2. 并行信息搜集:
   a. 使用 `SerpAPI` 工具，上网搜索出发件人问题的答案。
   b. 使用 `Simple Vector Store2` 工具，获取我设定的准确工作时间。
...
5. 格式化输出: 你必须将最终生成的邮件内容以一个严格的 JSON 格式输出:
   { "shouldReply": true, "subject": "Re: [原始邮件主题]", "body": "[...所有换行必须使用HTML的<br>标签]" }
```

又见 chapter4 的那条铁律——**提示词定协议，下游做解析**：要求输出 JSON、要求 `<br>` 换行，都是因为下游是"发 HTML 邮件"这个刚性消费者。

---

## 🧅 洋葱心：发送节点的三行表达式（第 84-88 行）

```json
"sendTo":  "={{ $('Gmail').item.json.From }}",
"subject": "=Re:  {{ $('Gmail').item.json.Subject }}",
"message": "={{ $json.output }}"
```

这三行浓缩了 n8n 数据流的全部语法：

- `$json` ——**上一个节点**（AI Agent）的输出，`$json.output` 就是 Agent 写好的信；
- `$('Gmail').item.json.X` ——**跨节点回溯引用**：发送时 Agent 已经把数据"洗"过一遍了，但收件人和原主题必须回到源头 Gmail 节点去拿。

一封邮件进来，经过"触发 → 思考（带工具、带记忆、带知识库）→ 回信"，全程没有一行需要部署的代码——但每一处提示词、每一条连线，都精确对应我们在 chapter4 手写过的某个 Python 类。

---

## 🎯 剥完之后：它在本章的位置

n8n 案例展示的是**"单智能体 + 能力插槽"**形态：整张图只有一个会思考的节点（AI Agent1），其余十个节点要么给它喂数据，要么给它装能力。

| 对比维度 | n8n 本例 | 其他三例 |
|---|---|---|
| 触发方式 | **定时轮询 Gmail**（无人值守自动化） | 其余三例都是用户对话触发 |
| 智能体形态 | 单 Agent 自主调工具（LangChain Tools Agent） | Coze 无 Agent；Dify/FastGPT 是多 Agent 分工 |
| RAG 用法 | 向量库包装成工具（retrieve-as-tool） | FastGPT 用独立的知识库搜索节点 |
| 独有机制 | main 边与 ai_* 边分离的"能力插槽" | Dify/FastGPT/Coze 的工具是节点内部配置项 |

看懂了"一个 Agent 插满插头"，再去看 Dify 和 FastGPT 的"多个 Agent 各管一摊"，就是从独奏到乐队的区别。
