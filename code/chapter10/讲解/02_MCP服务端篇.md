# MCP 服务端篇 剥洋葱式讲解

> 源文件：`code/chapter10/my_mcp_server.py`（239 行）、`code/chapter10/weather-mcp-server/server.py`（89 行）
> 一句话概括：**站到桌子另一边——用装饰器把普通 Python 函数发布成任何 MCP 客户端都能发现和调用的标准服务，甚至打包上线给全世界用。**

---

## 🧅 第一层（最外层）：这两个文件在干什么？

MCP 客户端篇里我们一直在"消费"服务器：`MCPClient(["python", "my_mcp_server.py"])` 连的那个神秘脚本，就是本篇第一个主角。它证明了一件事：**写一个 MCP 服务器不需要懂 JSON-RPC，不需要写网络代码——给函数加一行装饰器即可。**

- `my_mcp_server.py`：教学版服务器，用社区库 `fastmcp` 完整展示 MCP 的三种原语——**工具（Tools）、资源（Resources）、提示词（Prompts）**；
- `weather-mcp-server/server.py`：生产版服务器，用 HelloAgents 自带的 `MCPServer` 封装真实天气 API，配好 Dockerfile、pyproject.toml、smithery.yaml（这些打包文件本篇只略提），可发布到 Smithery 平台。

---

## 🧅 第二层：`my_mcp_server.py`——一行装饰器完成"函数 → 工具"

服务器的全部骨架只有两句（第 19 行、第 238 行）：

```python
mcp = FastMCP("MyCustomServer")
# ...中间全是业务函数...
mcp.run()
```

中间的业务函数长这样（第 24-36 行）：

```python
@mcp.tool()
def add(a: float, b: float) -> float:
    """
    加法计算器

    Args:
        a: 第一个数字
        b: 第二个数字
    """
    return a + b
```

`@mcp.tool()` 干的事远比看上去多——它把这个函数的三样东西自动翻译成 MCP 协议：

| Python 里写的 | 客户端 `list_tools` 里看到的 |
|---|---|
| 函数名 `add` | `tool['name']` |
| docstring | `tool['description']` |
| 类型标注 `a: float, b: float` | `tool['inputSchema']`（JSON Schema） |

客户端篇里打印的那些工具清单，源头就在这里。**docstring 和类型标注在 MCP 世界里不是注释，是接口契约**——LLM 靠 description 决定用不用你的工具，靠 inputSchema 决定怎么传参，写得含糊工具就等于不存在。

文件里共注册了 8 个工具：四则运算（add / subtract / multiply / divide，第 24-86 行）和文本处理（reverse_text / count_words / to_uppercase / to_lowercase，第 91-144 行）。`divide` 里的 `raise ValueError("除数不能为零")`（第 84-86 行）也值得看一眼：服务器抛的异常会通过协议传回客户端，变成客户端篇里 `try/except` 捕到的那个错误。

---

## 🧅 第三层：工具之外——资源与提示词（第 149-230 行）

MCP 不止有工具。这个文件还演示了另外两种原语：

**资源（Resource）——只读的"名词"**（第 149-164 行）：

```python
@mcp.resource("config://server")
def get_server_config() -> str:
    config = {"name": "MyCustomServer", "version": "1.0.0", ...}
    return json.dumps(config, ensure_ascii=False, indent=2)
```

资源用 URI 标识（`config://server`、`info://capabilities`），客户端按 URI 读取。工具是"做一件事"（动词），资源是"取一份数据"（名词）——语义分开后，客户端可以放心地预取资源而不担心副作用。

**提示词模板（Prompt）**（第 199-230 行）：

```python
@mcp.prompt()
def math_helper() -> str:
    return """你是一个数学计算助手。你可以使用以下工具：
- add(a, b): 计算两数之和
..."""
```

服务器连"怎么用我"的提示词都替客户端写好了。三种原语合起来，一个 MCP 服务器交付的是完整的能力包：**工具（手）+ 资源（资料）+ 提示词（说明书）**。

最后 `mcp.run()`（第 238 行）默认走 stdio 传输——这解释了客户端篇里为什么用一条 `["python", "my_mcp_server.py"]` 命令就能连上它：客户端把它拉成子进程，标准输入进、标准输出出。

---

## 🧅 第四层：`weather-mcp-server/server.py`——从玩具到真实服务

天气服务器不再返回写死的数据，而是真的去请求外部 API（第 22-40 行）：

```python
def get_weather_data(city: str) -> Dict[str, Any]:
    city_en = CITY_MAP.get(city, city)
    url = f"https://wttr.in/{city_en}?format=j1"
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    data = response.json()
    current = data["current_condition"][0]
    return {
        "city": city,
        "temperature": float(current["temp_C"]),
        ...
    }
```

三个生产级细节：

- **`CITY_MAP` 中英映射**（第 14-19 行）——wttr.in 只认英文城市名，服务器内部消化掉"北京 → Beijing"的转换，客户端和 LLM 可以直接说中文；
- **`timeout=10` + `raise_for_status()`**——外部依赖必须设超时、必须检查状态码；
- **数据清洗**——不把 wttr.in 的原始 JSON 原样吐给调用方，而是挑出温度、湿度、体感等字段重组成稳定结构。API 返回格式变了，只改这一个函数。

对外暴露的工具函数还包了一层错误处理（第 44-50 行）：

```python
def get_weather(city: str) -> str:
    try:
        weather_data = get_weather_data(city)
        return json.dumps(weather_data, ensure_ascii=False, indent=2)
    except Exception as e:
        return json.dumps({"error": str(e), "city": city}, ensure_ascii=False)
```

注意它**返回带 `error` 字段的 JSON 而不是抛异常**——因为最终读这个结果的是 LLM，一段结构化的错误描述比一个协议层异常更容易被模型理解并向用户解释。

---

## 🧅 洋葱心：注册与上线的五行（第 12、70-72、88 行）

```python
weather_server = MCPServer(name="weather-server", description="真实天气查询服务")

weather_server.add_tool(get_weather)
weather_server.add_tool(list_supported_cities)
weather_server.add_tool(get_server_info)

weather_server.run(transport="http", host=host, port=port)
```

这五行是服务端篇的最小本质：**创建服务器 → 注册函数 → 选传输方式运行**。与 `my_mcp_server.py` 对比能看出两个变化：

1. **注册方式**从装饰器换成了 `add_tool(函数)`——HelloAgents 的 `MCPServer` 风格，效果相同（函数签名和 docstring 照样被翻译成协议描述），业务函数保持纯净、可单独测试；
2. **传输方式**从默认 stdio 换成了 `transport="http"`，端口从环境变量 `PORT` 读取（第 77-78 行）——这是发布平台 Smithery 的硬性要求。同目录下的 `Dockerfile`、`pyproject.toml`、`smithery.yaml`、`PUBLISH_CHECKLIST.md` 就是围绕这一行 `run()` 的上线配套：把服务器装进容器、声明依赖、注册到市场。

同一份业务代码，改一个 `transport` 参数就从"本地子进程"变成"互联网服务"——客户端篇讲过的"协议与传输解耦"，在服务端这边合上了闭环。

---

## 🎯 剥完之后：它在本章的位置

服务端篇回答的问题是：**我的能力怎么让全世界的智能体用上？**答案是三步——写普通函数、注册到服务器、选传输方式运行。

| 消费方 | 怎么用到本篇的服务器 |
|---|---|
| `02_Connect2MCP.py` / `04_MCPTransport.py` | `MCPClient(["python", "my_mcp_server.py"])` 直连 |
| `05_UseMCPToolInAgent.py` | `MCPTool(server_command=["python", "my_mcp_server.py"])` 装进 Agent |
| `14_weather_mcp_server.py`（天气实战篇） | 就是 `weather-mcp-server/server.py` 的 stdio 本地版，同一份代码两种部署 |

MCP 的对称美在这两篇里完整呈现：客户端问 `list_tools`，服务端用装饰器/`add_tool` 应答；客户端 `call_tool` 传 JSON 参数，服务端普通函数收原生参数——**协议层的一切翻译工作，框架都替你做了。**
