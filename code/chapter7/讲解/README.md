# Chapter 7 代码剥洋葱讲解

对 `code/chapter7/` 下代码的逐层拆解。每篇文档都采用"剥洋葱"结构：从最外层的**这个文件在干什么**开始，一层层剥到最核心的**那几行关键代码**，最后回到全局，说明它在 HelloAgents 框架版图中的位置。

本章的主角是 `hello_agents` 框架本身：第四章我们**从零裸写**了 LLM 客户端、工具箱和三种智能体范式；本章这些东西都进了框架成为基类，我们改学第二种功夫——**站在框架上做扩展**：继承基类、只重写差异点。

## 推荐阅读顺序

先看 LLM 层的扩展（脑），再看两种工具形态（手），然后看两种 Agent 扩展（身），最后用测试脚本串起全部用法：

| 顺序 | 文档 | 源文件 | 主题 | 一句话 |
|---|---|---|---|---|
| 1 | [my_llm.md](my_llm.md) | `my_llm.py` | 扩展 LLM 层 | 继承 `HelloAgentsLLM`，旁路拦截自定义 ModelScope provider，其余交还父类 |
| 2 | [my_main.md](my_main.md) | `my_main.py` | 最小可运行入口 | 22 行验证继承成果：构造换了，`think()` 等能力全部白拿 |
| 3 | [my_calculator_tool.md](my_calculator_tool.md) | `my_calculator_tool.py` | 函数式工具 | AST 白名单安全计算器 + `register_function` 一行注册 |
| 4 | [my_advanced_search.md](my_advanced_search.md) | `my_advanced_search.py` | 类式工具 | 多搜索源探测、逐源降级，把实例方法注册成工具 |
| 5 | [my_simple_agent.md](my_simple_agent.md) | `my_simple_agent.py` | 扩展对话 Agent | 自创 `[TOOL_CALL:...]` 轻协议，让聊天 Agent 边聊边干活 |
| 6 | [my_react_agent.md](my_react_agent.md) | `my_react_agent.py` | 扩展 ReAct Agent | 第四章手写 ReAct 的框架化重生：只写提示词和循环，解析全靠继承 |
| 7 | [测试文件总览.md](测试文件总览.md) | `test_*.py` × 6 | 测试总览 | 六个演示脚本 = 框架六种用法的活文档（含两份留给读者的"结业作业"） |

## 一条主线

本章讲的是**如何扩展一个智能体框架**。三个扩展点构成"三板斧"，斧斧都是同一个动作——继承（或注册），只写差异：

> **扩展 LLM**（`my_llm.py`：拦截自己的 provider 分支，`else` 里一行 `super()` 放行其余）
> **扩展工具**（`my_calculator_tool.py` / `my_advanced_search.py`：函数也好、对象也好，`register_function` 面前一律平等）
> **扩展 Agent**（`my_simple_agent.py` / `my_react_agent.py`：重写 `run`，基建照单全收）

贯穿始终的一条框架设计铁律：**基类定契约，子类填血肉**——`Agent` 基类的抽象方法 `run(input_text)` 让四种范式对使用者只是同一次调用；`ToolRegistry` 的 `name + description + func` 三元组让任何函数三行变工具；而第四章的老朋友"提示词定协议，代码做解析"，在 `[TOOL_CALL:...]` 和 `Finish[...]` 里第四次、第五次落地。

> 前置阅读：chapter4 的讲解（`code/chapter4/讲解/`）逐层拆过 ReAct / Plan-and-Solve / Reflection 的裸写实现。对照阅读收益最大——同一个 ReAct，第四章你能看到每一根神经，第七章你能看到框架替你缝合了哪些神经。
