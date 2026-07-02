# MCP 客户端篇 剥洋葱式讲解

> 源文件：`code/chapter10/01_TestConnect.py`（24 行）、`02_Connect2MCP.py`（101 行）、`03_GitHubMCP.py`（33 行）、`04_MCPTransport.py`（83 行）、`05_UseMCPToolInAgent.py`（48 行）、`06_MultiAgentDocumentAssist.py`（134 行）
> 一句话概括：**从"三行连上一个 MCP 服务器"到"两个智能体借 MCP 工具协作产出报告"，六个文件层层递进，展示 MCP 协议如何把"接入外部服务"从写适配器变成填一条启动命令。**

---

## 🧅 第一层（最外层）：这组文件在干什么？

第七章的智能体每接一个外部服务（GitHub、数据库、文件系统），都要手写一个 Tool 类：处理 HTTP、鉴权、错误……写完还没法给别人复用。MCP（Model Context Protocol）要解决的就是这个**工具集成困境**：它规定了一套标准接口——服务端把能力描述成"工具清单"，客户端用统一的两个动作 `list_tools`（有什么）和 `call_tool`（用某个）就能消费任何服务器。

六个文件是一条难度递增的路径：

| 文件 | 角色 |
|---|---|
| `01_TestConnect.py` | 全章总览：MCP / A2A / ANP 三个工具各打一次招呼 |
| `02_Connect2MCP.py` | 底层 `MCPClient`：连接 → 发现 → 调用 → 容错，四个异步函数走完客户端全流程 |
| `03_GitHubMCP.py` | 连社区现成的 GitHub 服务器——"零代码接入外部 API"的实证 |
| `04_MCPTransport.py` | 传输层横评：Memory / Stdio / HTTP，同一协议跑在不同管道上 |
| `05_UseMCPToolInAgent.py` | 把 MCP 工具装进 `SimpleAgent`，让 LLM 自己决定调什么 |
| `06_MultiAgentDocumentAssist.py` | 综合实战：两个 Agent 各带一个 MCP 工具，分工产出研究报告 |

---

## 🧅 第二层：`01_TestConnect.py`——三种协议的最小可运行画像

这个文件放在全章开头，故意用最少的代码给三个协议各拍一张"证件照"（第 4-10 行）：

```python
mcp_tool = MCPTool()
result = mcp_tool.run({
    "action": "call_tool",
    "tool_name": "add",
    "arguments": {"a": 10, "b": 20}
})
print(f"MCP计算结果: {result}")  # 输出: 30.0
```

注意 `MCPTool()` 不带任何参数——它会启动一个**内置演示服务器**（提供 add、greet 等工具），所以这段代码不依赖网络就能跑通。`run()` 的入参格式值得记住：`action` 选动作、`tool_name` 选工具、`arguments` 传参数，这就是 MCPTool 对外的全部协议。

紧接着 ANP 注册并发现服务（第 13-21 行）、A2A 创建客户端（第 24-25 行）——各协议解决什么问题，读完这 25 行就有了直觉：**MCP 管"人用工具"，A2A 管"人与人对话"，ANP 管"在人群里找到对的人"。**

---

## 🧅 第三层：`02_Connect2MCP.py`——客户端生命周期的四步舞

这个文件绕过 MCPTool 包装，直接用底层 `MCPClient`，把客户端与服务器打交道的完整生命周期拆成四个异步函数。

**第一步：连接（第 7-17 行）**

```python
client = MCPClient([
    "npx", "-y",
    "@modelcontextprotocol/server-filesystem",
    "."  # 指定根目录
])

async with client:
    tools = await client.list_tools()
```

`MCPClient` 的构造参数是一条**启动命令**——它会把这条命令作为子进程拉起来，通过标准输入输出（stdio）与之交换 JSON-RPC 消息。`npx -y` 会自动下载社区包，也就是说：接入一个文件系统服务只需要写对一条命令行。`async with` 保证子进程随作用域结束被正确回收。

**第二步：发现（第 29-49 行）**

```python
tools = await client.list_tools()
for tool in tools:
    print(f"\n工具名称: {tool['name']}")
    print(f"描述: {tool.get('description', '无描述')}")
    if 'inputSchema' in tool:
        schema = tool['inputSchema']
```

每个工具自带三件套：`name`（叫什么）、`description`（干什么）、`inputSchema`（怎么传参，JSON Schema 格式）。这正是 MCP 的核心思想——**服务器自描述**。客户端（以及后面的 LLM）不需要提前知道服务器有什么，问一句 `list_tools` 就全知道了。

**第三步：调用（第 68-85 行）**

```python
result = await client.call_tool("read_file", {"path": "my_README.md"})
result = await client.call_tool("write_file", {
    "path": "output.txt",
    "content": "Hello from MCP!"
})
```

读文件、列目录、写文件——注意我们**一行文件操作代码都没写**，全部由服务器子进程代劳，客户端只负责按 schema 传参。仓库里的 `output.txt` 就是这一步的产物。

**第四步：容错（第 89-99 行）**

```python
try:
    result = await client.call_tool("read_file", {"path": "nonexistent.txt"})
except Exception as e:
    print(f"工具调用失败: {e}")
```

远程调用必然会失败（文件不存在、进程崩溃、超时），`try/except` 包住每次 `call_tool` 是使用姿势的一部分，不是可选项。

---

## 🧅 第四层：`03_GitHubMCP.py`——零代码接入 GitHub

传统做法接 GitHub 要写一个 `GitHubTool` 类去封装 REST API。MCP 做法（第 12-14 行）：

```python
github_tool = MCPTool(
    server_command=["npx", "-y", "@modelcontextprotocol/server-github"]
)
```

一条命令换掉整个适配器。搜索仓库的调用（第 23-31 行）参数 `query` / `page` / `perPage` 从哪来？从 `list_tools` 返回的 `inputSchema` 来——又一次体现"服务器自描述"。唯一的准备工作是设置环境变量 `GITHUB_PERSONAL_ACCESS_TOKEN`（文件头注释给了 Windows/Linux 两种写法），**鉴权归服务器管，客户端不碰密钥逻辑**。

---

## 🧅 第五层：`04_MCPTransport.py`——同一协议，三种管道

MCP 消息格式（JSON-RPC 2.0）与消息怎么运输是解耦的。这个文件把常见传输方式排成一列（第 3-21 行）：

```python
mcp_tool = MCPTool()                                            # Memory：内置服务器，测试用
mcp_tool = MCPTool(server_command=["python", "my_mcp_server.py"])  # Stdio：本地子进程
mcp_tool = MCPTool(server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."])  # Stdio：社区服务器
```

规律一眼可见：**换传输 = 换构造参数，`run()` 的用法一个字不用改。**远程 HTTP 场景则直接用底层客户端（第 69-81 行）：

```python
client = MCPClient("http://api.example.com/mcp")
async with client:
    tools = await client.list_tools()
```

`MCPClient` 收到字符串 URL 就走 HTTP，收到命令列表就走 stdio——一个构造函数，靠参数类型自动分流。文件末尾的注释也说清了分工：MCPTool 适合 Stdio/Memory 的同步场景，远程传输建议直接用 MCPClient。

---

## 🧅 第六层：`05_UseMCPToolInAgent.py`——把工具递给 LLM

前面都是"人写代码调用工具"，这一层把决策权交给智能体（第 8-17 行）：

```python
agent = SimpleAgent(name="助手", llm=HelloAgentsLLM())
mcp_tool = MCPTool()
agent.add_tool(mcp_tool)

response = agent.run("计算 123 + 456")   # 智能体会自动调用add工具
```

用户说自然语言，LLM 从工具描述里认出 `add` 并自己拼好 `arguments`——第四章 ReAct 循环里"提示词定协议"的老朋友，在这里由框架接管了。

接多个服务器时有一个工程细节（第 26-40 行）：

```python
fs_tool = MCPTool(
    name="filesystem",      # 指定唯一名称
    server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]
)
custom_tool = MCPTool(
    name="custom_server",   # 使用不同的名称
    server_command=["python", "my_mcp_server.py"]
)
```

**每个 MCPTool 必须起不同的 `name`**，否则两个服务器的工具在 Agent 的注册表里会互相覆盖——注释里专门用"重要"标注了这一点。

---

## 🧅 洋葱心：`06_MultiAgentDocumentAssist.py` 的双 Agent 流水线（第 37-41、64-68、82-105 行）

```python
github_tool = MCPTool(name="gh", server_command=["npx", "-y", "@modelcontextprotocol/server-github"])
github_searcher.add_tool(github_tool)

fs_tool = MCPTool(name="fs", server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."])
document_writer.add_tool(fs_tool)

search_results = github_searcher.run(search_task)
report_task = f"""根据以下GitHub搜索结果，生成一份Markdown格式的研究报告：\n\n{search_results}\n..."""
report_content = document_writer.run(report_task)
```

这几行是整个 MCP 客户端篇的收束：

1. **能力即插件**——搜索专家插 GitHub 服务器、写作专家插文件系统服务器，各自的 system_prompt（第 27-33、51-60 行）只描述职责，不描述 API；
2. **Agent 之间用字符串接力**——Agent1 的输出 `search_results` 被 f-string 塞进 Agent2 的任务里，这是最朴素的多 Agent 协作（没有任何协议，纯提示词拼接），也正是为后面 A2A 篇埋的伏笔：*当协作需要跨进程、跨机器时，f-string 就不够用了*；
3. **最后一步不信任 LLM**——保存 `report.md` 用的是普通 Python `open()`（第 116-117 行）而不是让 Agent 调文件工具，并且写完还要 `os.path.getsize` 验证。提示词里甚至明确要求"不要使用工具保存"（第 60 行）——凡是必须确定发生的事，用代码；凡是需要理解和生成的事，用 LLM。

仓库里的 `report.md`、`a2a_document_*.md` 就是这类流水线的运行产物，看一眼就知道成品长什么样。

---

## 🎯 剥完之后：它在本章的位置

MCP 客户端篇回答了本章第一个问题：**智能体怎么用上全世界的工具？**答案是把"写适配器"变成"填启动命令"，把"读 API 文档"变成"调 list_tools"。

| 后续文档 | 与本篇的关系 |
|---|---|
| MCP 服务端篇 | 这里连的 `my_mcp_server.py` 是怎么写出来的 |
| A2A 基础篇 | `06` 里 f-string 式的 Agent 接力，升级成真正的网络协议 |
| 天气智能体实战篇 | 客户端 + 服务端 + Agent 三件套的完整落地 |
