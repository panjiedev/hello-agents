# my_calculator_tool.py 剥洋葱式讲解

> 源文件：`code/chapter7/my_calculator_tool.py`（62 行）
> 一句话概括：**用 AST 白名单实现一个"绝不执行任意代码"的安全计算器，再用 `register_function` 一行把普通函数变成框架工具。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

给 Agent 添加自定义工具，最低成本的路径是什么？本文件给出答案：**写一个普通的 Python 函数，然后注册进 `ToolRegistry`**。全文分两块：

1. `my_calculate(expression)` ——一个接收字符串、返回字符串的纯函数（第 7-31 行）；
2. `create_calculator_registry()` ——把这个函数包装成 Agent 可调用的工具（第 51-62 行）。

注意函数的输入输出都是**字符串**：这不是偶然，而是 LLM 工具的通用契约——LLM 只会说话（产出文本），也只能听话（读入文本），所以工具的接口天然就是"文本进、文本出"。

---

## 🧅 第二层：为什么不用一行 `eval()`？（第 13-24 行）

算一个 `"2 + 3"`，最偷懒的写法是 `eval(expression)`。但 LLM 生成的表达式本质是**不可信输入**——万一模型（或诱导模型的用户）产出 `__import__('os').system('rm -rf /')` 呢？`eval` 会照单全收。

本文件的方案是 **AST 白名单**：

```python
operators = {
    ast.Add: operator.add,      # +
    ast.Sub: operator.sub,      # -
    ast.Mult: operator.mul,     # *
    ast.Div: operator.truediv,  # /
}
functions = {
    'sqrt': math.sqrt,
    'pi': math.pi,
}
```

先用 `ast.parse(expression, mode='eval')`（第 27 行）把字符串解析成语法树——**只解析，不执行**；再自己遍历这棵树，节点类型必须命中白名单才求值。四则运算之外的任何东西（属性访问、import、幂运算……）在白名单里查无此人，直接算不出来。

---

## 🧅 第三层：`_eval_node`——递归下降求值器（第 33-49 行）

```python
if isinstance(node, ast.Constant):
    return node.value
elif isinstance(node, ast.BinOp):
    left = _eval_node(node.left, operators, functions)
    right = _eval_node(node.right, operators, functions)
    op = operators.get(type(node.op))
    return op(left, right)
```

表达式 `sqrt(16) + 2 * 3` 解析后是一棵树：根是 `+`，左子树是函数调用 `sqrt(16)`，右子树是 `2 * 3`。`_eval_node` 对树做递归：

- **叶子**（`ast.Constant`）：数字本身，直接返回；
- **二元运算**（`ast.BinOp`）：先递归算出左右子树，再查表拿运算函数；
- **函数调用**（`ast.Call`）：函数名必须在 `functions` 白名单里，递归求参后调用。

这就是编译原理里"递归下降求值"的最小实现——三十行代码，胜过所有对 `eval` 的提心吊胆。外层的 `try/except`（第 26-31 行）保证任何非法输入都只返回一句"计算失败"，**工具永远不该向 Agent 抛异常，报错也是一种合法的 Observation**。

---

## 🧅 洋葱心：三行注册（第 56-60 行）

```python
registry.register_function(
    name="my_calculator",
    description="简单的数学计算工具，支持基本运算(+,-,*,/)和sqrt函数",
    func=my_calculate
)
```

普通函数到"Agent 工具"的距离就是这一次注册。三个参数各司其职：

- `name`——Agent 调用时的句柄（`registry.execute_tool("my_calculator", "2+3")`）；
- `description`——**写给 LLM 看的**。它会被拼进提示词的"可用工具"清单，是 LLM 决定"什么时候用这个工具"的唯一依据，所以要写清能力边界（"支持 +,-,*,/ 和 sqrt"）；
- `func`——真正干活的函数。

对照第四章 `tools.py`：当年我们手写了 `ToolExecutor` 类来维护 `name → (func, description)` 字典；本章这个字典进了框架，我们只剩下"往里塞东西"的一行调用。

---

## 🎯 剥完之后：它在本章的位置

`my_calculator_tool.py` 展示了框架扩展三板斧的第二斧——**函数式工具**（最轻量的工具形态）：

| 工具形态 | 文件 | 适用场景 |
|---|---|---|
| 函数式（`register_function`） | 本文件 | 无状态、单一功能 |
| 类式（实例方法注册） | `my_advanced_search.py` | 需要初始化、持有状态、多数据源 |

它在 `test_my_calculator.py` 中被单独验证，也可以塞进 `MySimpleAgent` / `MyReActAgent` 的 `ToolRegistry` 成为它们的"手"。安全求值的思想则值得带走：**凡是执行 LLM 产出的代码/表达式，白名单永远优于黑名单，解析永远优于 eval。**
