# tools.py 剥洋葱式讲解

> 源文件：`code/chapter4/tools.py`（110 行）
> 一句话概括：**给智能体装上"手"——一个真实的 Google 搜索工具（SerpApi），加一个可插拔的工具箱 `ToolExecutor`。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

LLM 只有"脑"没有"手"：知识停留在训练截止日，也无法查询实时信息。工具（Tool）就是智能体的手。这个文件提供两样东西：

1. **`search(query)`**——一个能真正联网搜 Google 的函数（通过 SerpApi 服务）；
2. **`ToolExecutor`**——一个工具登记簿，智能体通过它按名字查找并调用工具。

直接运行本文件，会注册 Search 工具并实际搜索一次"英伟达最新的GPU型号是什么"，演示完整的"注册 → 查找 → 调用 → 观察"链路。

---

## 🧅 第二层：`search` 函数——真实世界的搜索（第 9-49 行）

```python
params = {
    "engine": "google",
    "q": query,
    "api_key": api_key,
    "gl": "cn",     # 国家代码
    "hl": "zh-cn",  # 语言代码
}
client = SerpApiClient(params)
results = client.get_dict()
```

为什么不直接爬 Google？因为会被反爬机制拦截，且结果页是复杂的 HTML。SerpApi 是中间商：你发关键词，它返回**结构化的 JSON**。`gl` / `hl` 两个参数把搜索结果本地化成中文——面向中文问题时结果质量差异很大。

函数开头的 `os.getenv("SERPAPI_API_KEY")` 检查沿用了 fail-fast 思路，但注意它返回的是**一个错误字符串而不是抛异常**——这是有意为之：错误信息会作为 Observation 喂回给 LLM，让智能体自己"看到"失败并调整策略（详见 ReAct 讲解）。

---

## 🧅 第三层：智能解析——四级答案瀑布（第 32-46 行）

这是 `search` 里最有含金量的部分。Google 返回的 JSON 里可能有很多种"答案"，按**直接程度**排出优先级瀑布：

```python
if "answer_box_list" in results: ...        # ① 答案框列表：最直接
if "answer_box" in results and ...: ...     # ② 单个答案框（如"珠峰高多少"直接给数字）
if "knowledge_graph" in results and ...: ...# ③ 知识图谱描述（如搜人名给简介）
if "organic_results" in results and ...: ...# ④ 兜底：前三条普通搜索结果的标题+摘要
```

为什么这个顺序如此重要？因为**搜索结果最终要塞进 LLM 的提示词里**：

- 越靠前的来源越精炼，token 越少、噪声越小，模型越不容易被带偏；
- 兜底的有机结果特意只取前 3 条、且只要标题和摘要（snippet），编号 `[1] [2] [3]` 方便模型引用。

一句话：**工具的输出质量 = 智能体的观察质量**。这个瀑布就是在为 LLM 做"信息降噪"。

---

## 🧅 洋葱心：`ToolExecutor`——25 行的插件系统（第 53-83 行）

```python
class ToolExecutor:
    def __init__(self):
        self.tools: Dict[str, Dict[str, Any]] = {}

    def registerTool(self, name, description, func): ...
    def getTool(self, name) -> callable: ...
    def getAvailableTools(self) -> str: ...
```

它本质上就是一个字典：`{工具名: {"description": 描述, "func": 函数}}`。但这个小字典解决了智能体架构的一个根本问题——**LLM 怎么知道自己能用什么工具、程序怎么执行 LLM 选的工具？** 答案是同一份注册信息的两个面向：

- **`getAvailableTools()` 面向 LLM**：把所有工具拼成 `- Search: 一个网页搜索引擎。当你需要回答…` 的文本清单，塞进提示词。**描述（description）就是工具的"使用说明书"，是写给模型看的，写得好坏直接决定模型会不会在正确的时机选对工具**——这是新手最容易轻视的一点。
- **`getTool(name)` 面向程序**：模型输出 `Search[华为最新手机]` 后，程序按名字取出真函数来执行。

注意 `getTool` 的写法：`self.tools.get(name, {}).get("func")`——查不到不抛异常，返回 `None`，让调用方决定如何处理"模型编造了一个不存在的工具"（ReAct 里会把这个错误作为 Observation 喂回去）。

这就是**注册表模式（Registry Pattern）**：新增工具零侵入——写个函数、`registerTool` 一行注册，智能体的提示词自动多出一条工具说明。今天所有 Agent 框架（LangChain 的 Tool、OpenAI 的 function calling、MCP 协议）的工具机制，内核都是这个字典的豪华版。

---

## 🎯 剥完之后：它在本章的位置

`tools.py` 和 `llm_client.py` 是本章的两块基石——**一个是脑，一个是手**：

```
ReActAgent
 ├── llm_client.think()        → 决定"该做什么"（Thought / Action）
 └── tool_executor.getTool()() → 实际"做出来"（Observation）
```

`ReAct.py` 会把这两块拼成完整的"思考-行动"循环，那才是本章真正的主角。
