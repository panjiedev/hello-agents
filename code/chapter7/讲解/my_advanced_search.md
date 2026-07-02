# my_advanced_search.py 剥洋葱式讲解

> 源文件：`code/chapter7/my_advanced_search.py`（132 行）
> 一句话概括：**一个"类式"工具的范本——启动时探测可用搜索源（Tavily / SerpApi），运行时逐源尝试、自动降级，最后把实例方法注册为框架工具。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

`my_calculator_tool.py` 展示了最简单的工具：一个无状态函数。但真实世界的工具往往需要**状态**——要持有 API 客户端、要记住哪些数据源可用、要在多个后端之间做选择。这时函数就不够用了，得上类：

```python
class MyAdvancedSearchTool:
    def __init__(self):
        self.search_sources = []      # 状态：可用的搜索源列表
        self._setup_search_sources()  # 启动时探测

    def search(self, query: str) -> str:
        # 运行时逐源尝试
```

它整合两个搜索后端：Tavily（AI 搜索，能直接给答案）和 SerpApi（Google 搜索代理），谁可用用谁，谁先成功用谁。对照第四章 `tools.py` 里的单一 SerpApi 搜索函数，本文件是它的"高可用版"。

---

## 🧅 第二层：启动探测——能力是"发现"出来的（第 18-42 行）

```python
if os.getenv("TAVILY_API_KEY"):
    try:
        from tavily import TavilyClient
        self.tavily_client = TavilyClient(api_key=os.getenv("TAVILY_API_KEY"))
        self.search_sources.append("tavily")
        print("✅ Tavily搜索源已启用")
    except ImportError:
        print("⚠️ Tavily库未安装")
```

每个搜索源要过**两道关**才能进 `search_sources` 名单：

1. **配置关**——环境变量里有没有 API key（`os.getenv`）；
2. **依赖关**——对应的第三方库装没装（`try: import ... except ImportError`）。

注意 `import` 写在函数体内而不是文件顶部——这叫**延迟导入**：没装 `tavily` 库的用户照样能 import 本文件，只是少一个搜索源，而不是整个模块崩掉。工具的可用性不是写死的，是启动时对环境"体检"出来的。

---

## 🧅 第三层：失败也要说人话（第 50-59 行）

```python
if not self.search_sources:
    return """❌ 没有可用的搜索源，请配置以下API密钥之一：

1. Tavily API: 设置环境变量 TAVILY_API_KEY
   获取地址: https://tavily.com/
..."""
```

一个搜索源都没有时，`search` 不抛异常、不返回空串，而是返回一份**带获取地址的配置指南**。别忘了这个返回值的读者有两个：屏幕前的人，和调用工具的 LLM——LLM 读到这段话，就能在最终回答里告诉用户"请先配置 API 密钥"，而不是胡编一个搜索结果。**错误信息的质量，直接决定 Agent 兜底行为的质量。**

---

## 🧅 洋葱心：逐源降级循环（第 64-80 行）

```python
for source in self.search_sources:
    try:
        if source == "tavily":
            result = self._search_with_tavily(query)
            if result and "未找到" not in result:
                return f"📊 Tavily AI搜索结果：\n\n{result}"
        elif source == "serpapi":
            result = self._search_with_serpapi(query)
            if result and "未找到" not in result:
                return f"🌐 SerpApi Google搜索结果：\n\n{result}"
    except Exception as e:
        print(f"⚠️ {source} 搜索失败: {e}")
        continue

return "❌ 所有搜索源都失败了，..."
```

这十几行是整个文件的灵魂，实现了三层容错：

1. **优先级**——按 `search_sources` 的顺序尝试（Tavily 优先，它有 AI 直接答案）；
2. **结果质检**——不仅要"没抛异常"，还要"结果有货"（`"未找到" not in result`），空转的成功不算成功；
3. **异常降级**——某一源崩了（网络、限额、密钥失效）就 `continue` 换下一家，全军覆没才认输。

这就是分布式系统里 **fallback（故障转移）** 模式的微缩版。两个 `_search_with_*` 私有方法（第 82-116 行）则负责把两家 API 各不相同的返回结构，统一格式化成"编号 + 标题 + 摘要"的纯文本——**对外一个声音说话，差异全部消化在内部**。

---

## 🧅 第五层：把"方法"注册成工具（第 118-132 行）

```python
search_tool = MyAdvancedSearchTool()

registry.register_function(
    name="advanced_search",
    description="高级搜索工具，整合Tavily和SerpAPI多个搜索源，...",
    func=search_tool.search
)
```

点睛之笔：注册的不是裸函数，而是**实例的绑定方法** `search_tool.search`。Python 的绑定方法自带 `self`，所以注册表调用它时，实例里探测好的 `search_sources`、建好的 `tavily_client` 全都在场。这样类式工具就通过和函数式工具**完全相同的接口**接入了框架——注册表根本不知道（也不需要知道）背后是函数还是对象。

---

## 🎯 剥完之后：它在本章的位置

`my_advanced_search.py` 是工具开发的"进阶示范"，和计算器构成一对教学样本：

| | `my_calculator_tool.py` | `my_advanced_search.py` |
|---|---|---|
| 形态 | 纯函数 | 类 + 绑定方法 |
| 状态 | 无 | 搜索源列表、API 客户端 |
| 容错 | try/except 兜底 | 探测 + 质检 + 逐源降级 |
| 教的事 | 安全求值、最小注册 | 多源整合、优雅降级 |

它由 `test_advanced_search.py` 验证，也可作为 `MyReActAgent` 的"眼睛"接入。带走的设计心法：**工具要把混乱的外部世界（不同 API、不同故障）翻译成 LLM 能稳定消化的一段纯文本。**
