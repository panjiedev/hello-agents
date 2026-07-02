# my_llm.py 剥洋葱式讲解

> 源文件：`code/chapter7/my_llm.py`（41 行）
> 一句话概括：**继承框架的 `HelloAgentsLLM`，为它"外挂"一个自定义的 ModelScope provider——只处理自己关心的分支，其余全部交还父类。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

第四章我们从零手写了一个 `HelloAgentsLLM` 客户端（`code/chapter4/llm_client.py`），73 行代码只干一件事：调 LLM。到了本章，这个类已经"毕业"进了 `hello_agents` 框架，功能大大增强——内置了 openai / deepseek / qwen / modelscope / ollama / vllm 等十来种 provider 的自动检测和凭证解析。

那如果框架内置的 provider 逻辑不合我意，比如我想给 ModelScope 定制自己的默认模型、默认地址呢？答案不是改框架源码，而是**继承 + 重写**：

```python
class MyLLM(HelloAgentsLLM):
    def __init__(self, ..., provider="auto", **kwargs):
        if provider == "modelscope":
            # 我自己的逻辑
        else:
            super().__init__(...)   # 其余情况原样交还父类
```

整个文件只重写了 `__init__` 一个方法——`think`、`invoke`、`stream_invoke` 等真正干活的方法一行没碰，全部继承。**这就是框架化开发和第四章裸写的根本区别：只写差异，不写重复。**

---

## 🧅 第二层：拦截自己关心的分支（第 17-19 行）

```python
if provider == "modelscope":
    print("正在使用自定义的 ModelScope Provider")
    self.provider = "modelscope"
```

`__init__` 的第一件事是判断 `provider` 参数。只有等于 `"modelscope"` 时才走自定义路径；其他任何值（包括默认的 `"auto"`）都落进第 38-40 行的 `else`，原封不动调用 `super().__init__(...)`，让框架的自动检测机制接管。

这是一种"**旁路拦截**"模式：像网络代理一样，只截获目标流量，其余直接放行。好处是升级框架时自定义逻辑不会被覆盖，父类新增的 provider 也自动可用。

---

## 🧅 第三层：自定义分支里做了父类同款的四件事（第 21-36 行）

```python
self.api_key = api_key or os.getenv("MODELSCOPE_API_KEY")
self.base_url = base_url or "https://api-inference.modelscope.cn/v1/"

if not self.api_key:
    raise ValueError("ModelScope API key not found. ...")

self.model = model or os.getenv("LLM_MODEL_ID") or "Qwen/Qwen2.5-VL-72B-Instruct"
```

眼熟吗？这正是第四章 `llm_client.py` 里的"配置瀑布"再现，只是链条更长了：**显式传参 > 环境变量 > 硬编码默认值**。`base_url` 和 `model` 都有兜底默认值，所以用户最少只需设一个 `MODELSCOPE_API_KEY` 环境变量。缺了 api_key 则立即 `raise`（fail fast），把配置错误暴露在构造时刻而不是第一次调用时。

```python
self._client = OpenAI(api_key=self.api_key, base_url=self.base_url, timeout=self.timeout)
```

第 36 行是关键的"接口契约"：把创建好的 OpenAI 客户端赋给 `self._client`——**属性名必须和父类保持一致**，因为继承来的 `think` / `invoke` 内部用的就是 `self._client`。子类只要把这几个属性（`provider`、`model`、`_client`……）按父类的约定摆好，父类的方法就能在子类实例上正常运转。

---

## 🧅 洋葱心：一行 `super()`（第 38-40 行）

```python
else:
    # 如果不是 modelscope, 则完全使用父类的原始逻辑来处理
    super().__init__(model=model, api_key=api_key, base_url=base_url, provider=provider, **kwargs)
```

整个文件的灵魂其实是这一行。它意味着 `MyLLM` 不是父类的**替代品**，而是父类的**超集**：

- 传 `provider="modelscope"` → 走我的定制逻辑；
- 传 `provider="openai"` / `"deepseek"` / 不传 → 和直接用 `HelloAgentsLLM` 一模一样。

注意 `**kwargs` 也被完整转发——父类未来新增任何构造参数，这个子类都不用改。这种"透传"写法是编写健壮子类的标准姿势。

---

## 🎯 剥完之后：它在本章的位置

`my_llm.py` 是本章"扩展框架三板斧"的第一斧——**扩展 LLM 层**：

| 扩展点 | 文件 | 手法 |
|---|---|---|
| LLM provider | `my_llm.py` | 继承 `HelloAgentsLLM`，拦截自定义分支 |
| 工具 | `my_calculator_tool.py` / `my_advanced_search.py` | `registry.register_function` 注册 |
| Agent 行为 | `my_simple_agent.py` / `my_react_agent.py` | 继承 Agent 基类，重写 `run` |

它的直接用户是 `my_main.py`——那 22 行代码会证明：构造函数换了，`think()` 等一切能力照旧白拿。对照第四章你会发现一个轮回：当年手写的 `HelloAgentsLLM` 如今成了被继承的框架基类，**你写的代码变成了别人的地基**——这正是"从用框架到写框架"的完整闭环。
