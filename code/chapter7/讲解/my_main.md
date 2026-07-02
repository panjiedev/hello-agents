# my_main.py 剥洋葱式讲解

> 源文件：`code/chapter7/my_main.py`（22 行）
> 一句话概括：**用 22 行代码验证 `MyLLM` 的继承成果——只换了构造方式，`think()` 等全部能力从父类白拿。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

它是 `my_llm.py` 的"验收脚本"：实例化自定义的 `MyLLM`，发一条消息，看流式回答能否正常吐出来。全文没有一个类、没有一个函数定义，就是最朴素的四步：**加载配置 → 建客户端 → 发消息 → 消费响应**。

```python
llm = MyLLM(provider="modelscope")
messages = [{"role": "user", "content": "你好，请介绍一下你自己。"}]
response_stream = llm.think(messages)
```

麻雀虽小，但有两个值得剥开的细节。

---

## 🧅 第二层：`provider="modelscope"` 触发的是谁的代码？（第 3、9 行）

```python
from my_llm import MyLLM  # 注意：这里导入我们自己的类

llm = MyLLM(provider="modelscope")
```

第 3 行的注释是作者特意留的路标：导入的是**我们自己的** `MyLLM`，不是框架的 `HelloAgentsLLM`。传入 `provider="modelscope"` 恰好命中 `my_llm.py` 第 17 行的 `if` 分支——运行时你会看到那句 `"正在使用自定义的 ModelScope Provider"`，这就是自定义逻辑生效的证据。

而第 15 行的 `llm.think(messages)`——`MyLLM` 里根本没定义 `think` 方法，Python 沿着继承链找到父类 `HelloAgentsLLM.think`，它内部用的 `self._client` 正是子类构造时按约定摆好的那个 ModelScope 客户端。**子类备料，父类掌勺**，一顿饭就做成了。

---

## 🧅 洋葱心：一个必须被"喝干"的生成器（第 15-22 行）

```python
response_stream = llm.think(messages)

print("ModelScope Response:")
for chunk in response_stream:
    # chunk在my_llm库中已经打印过一遍，这里只需要pass即可
    pass
```

最容易让初学者困惑的就是这个 `for ... pass`。关键在于：**框架版 `think()` 返回的不是字符串，而是一个流式生成器（`Iterator[str]`）**——这和第四章手写版不同，那个版本内部收集完直接返回完整字符串。

生成器是惰性的：不迭代，网络请求就不会真正走完。而框架的 `think` 在产出每个 chunk 时已经顺手打印到了屏幕（打字机效果），所以这里的循环体什么都不用做，`pass` 即可——**循环的意义不在处理 chunk，而在"驱动"整条流把话说完**。删掉这个循环，你只会看到标题 `ModelScope Response:` 后面一片空白。

---

## 🎯 剥完之后：它在本章的位置

`my_main.py` 是本章最小的可运行入口，也是"继承式扩展"的现场证明：

| 代码行数 | 自己写的 | 白拿的 |
|---|---|---|
| 22 行 | 指定 `provider="modelscope"` | 流式调用、打字机输出、消息协议、异常处理 |

第四章要享受同样的功能得先写 73 行客户端；本章只需继承 + 一个构造参数。后面的 `test_*.py` 系列会把这个"白拿"故事推广到 Agent 层：`SimpleAgent`、`ReActAgent` 同样是几行实例化就能跑起来。
