# 05_terminal_tool_examples 剥洋葱式讲解

> 源文件：`code/chapter9/05_terminal_tool_examples.py`（162 行）
> 一句话概括：**TerminalTool 给智能体一把"带保险栓的瑞士军刀"——用白名单 shell 命令即时探索文件系统，按需拉取上下文，而不是预先把所有文件塞进窗口或向量库。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

RAG 的思路是"预先索引"：先把文档切块、向量化、入库，查询时召回。但很多场景根本等不起也不值得建索引——看一眼日志尾部、数一下 CSV 行数、grep 一遍 TODO。这些需要的是**即时（Just-in-Time）上下文**：现场执行一条命令，只把命令输出（往往几十行）放进上下文。

本文件不定义新类，而是用五个 `demo_*` 函数展示 TerminalTool 的五种打开方式：探索式导航、数据文件分析、日志分析、代码库分析、安全特性。它是本章唯一**不需要 LLM 就能跑**的两个示例之一（另一个是 03），适合最先运行。

---

## 🧅 第二层：探索式导航——从粗到细的四步（第 25-44 行）

```python
terminal = TerminalTool(workspace=str(SCRIPT_DIR))

result = terminal.run({"command": "ls -la"})                              # 1. 看目录
result = terminal.run({"command": "ls -la *.py"})                         # 2. 缩小到 .py
result = terminal.run({"command": "find . -name '*codebase_maintainer.py'"})  # 3. 定位文件
result = terminal.run({"command": "head -n 20 codebase_maintainer.py"})   # 4. 只看前20行
```

四步暗含一个上下文经济学原则：**每一步只花最少的 token 换取下一步的方向**。尤其第 4 步用 `head -n 20` 而不是 `cat`——看开头的 docstring 和 import 就够判断"这文件是不是我要找的"，全文 476 行留到确认后再读。这正是后面 `codebase_maintainer.py` 里 Agent 自主探索时该学会的"节流"姿势。

注意接口和 NoteTool 完全同构：`run({"command": ...})`——一个 dict、一个动作字段，方便被 Function Calling 驱动。

---

## 🧅 第三层：管道即"上下文预处理器"（第 68、87 行）

数据分析场景里最漂亮的一行（第 68 行）：

```python
result = terminal.run({"command": "tail -n +2 sales_2024.csv | cut -d',' -f3 | sort | uniq -c"})
```

拆开看：跳过表头（`tail -n +2`）→ 取第 3 列类别（`cut`）→ 排序（`sort`）→ 分组计数（`uniq -c`）。40 多行的 CSV 进去，出来只有两三行"类别: 数量"。

日志分析场景同理（第 87 行）：

```python
"grep ERROR app.log | awk '{print $4}' | sort | uniq -c | sort -rn"
```

一天的日志进去，出来是按频次排序的错误类型分布表。

这两条管道揭示了 TerminalTool 的深层价值：**Unix 管道是免费的上下文压缩器**。与其把原始文件灌给 LLM 让它自己数（费 token、还容易数错），不如让确定性的文本工具先做聚合，LLM 只消费聚合结果。计算下沉给 shell，判断留给模型——各干各擅长的事。

代码库分析场景（第 96-117 行）把同样的思路用在代码上：`wc -l` 统计规模、`grep -rn 'TODO'` 列问题清单、`grep -rn 'def process_data'` 定位定义——这三板斧就是第 06 篇里 Agent"分析代码质量"时实际会用的招式。

---

## 🧅 第四层：安全演示——故意干坏事（第 120-141 行）

```python
result = terminal.run({"command": "rm -rf /"})        # 1. 危险命令
result = terminal.run({"command": "cat /etc/passwd"}) # 2. 越权读文件
result = terminal.run({"command": "cd ../../../etc"}) # 3. .. 逃逸
```

demo 专门写了三次"攻击"，全部被挡下。背后是四层防线（实现在 `hello_agents` 包里，详见教材 9.5.1 节）：

1. **命令白名单**——只放行只读命令（ls/cat/grep/find/awk/sed/wc...），`rm` 根本不在名单上；
2. **工作目录沙箱**——所有路径必须落在 `workspace` 内，`resolve()` 后用 `relative_to(workspace)` 校验，`..` 逃逸无效；
3. **超时控制**——防止命令挂死（`timeout` 参数）；
4. **输出大小限制**——防止 `cat 巨型文件` 撑爆内存和上下文。

给智能体递刀之前先装保险栓——**能力和约束必须同时交付**，这是所有"让 LLM 执行命令"类工具的第一设计原则。

---

## 🧅 洋葱心：workspace 这一个参数（第 25、54、78、102、126 行）

```python
terminal = TerminalTool(workspace=str(SCRIPT_DIR / "data"))
terminal = TerminalTool(workspace=str(SCRIPT_DIR / "logs"))
terminal = TerminalTool(workspace=str(SCRIPT_DIR / "codebase"))
```

五个场景创建了五个实例，唯一的区别就是 `workspace`。这一个参数同时干了两件事：

1. **划定权限边界**——沙箱的围墙就砌在这里；
2. **设定探索起点**——`ls` 看到的、`find .` 搜到的，都是这个目录的世界。

对智能体而言，`workspace` 就是"你被允许感知的世界范围"。`codebase_maintainer.py` 里 `TerminalTool(workspace=codebase_path)` 一行，就把 Agent 的手脚限定在了被维护的代码库之内——**最小权限原则，用一个构造参数落地**。

---

## 🎯 剥完之后：它在本章的位置

至此本章三种上下文来源集齐，各管一种时效：

| 工具 | 上下文类型 | 时效 | 类比 |
|---|---|---|---|
| ContextBuilder | 装配层 | 每轮重建 | 编辑部（决定版面）|
| NoteTool | 外部记忆 | 跨天/跨会话 | 工作日志 |
| **TerminalTool** | **即时检索** | **当下这一秒** | **现场勘查** |

在 `codebase_maintainer.py` 里，TerminalTool 被注册进 ToolRegistry 交给 Agent 自主调用——Agent 想看什么文件、跑什么统计，自己现场决定。**JIT 上下文的精髓：不预测模型需要什么，而是给它一双（戴着手铐的）手，让它自己去拿。**
