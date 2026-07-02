# ANP 篇 剥洋葱式讲解

> 源文件：`code/chapter10/11_ANPInit.py`（51 行）、`13_ANPLoadBalancing.py`（34 行）、`12_ANPTaskDistribution.py`（80 行）
> 一句话概括：**当智能体多到记不住 URL，就需要一个"电话簿"——ANP（Agent Network Protocol）用注册中心 + 元数据实现服务发现、负载均衡，甚至让 LLM 亲自当调度员。**

---

## 🧅 第一层（最外层）：这组文件在干什么？

A2A 应用篇结尾留了个问题：客服系统硬编码了 `http://localhost:6000` 和 `6001`。规模一大这就崩了——节点动态上下线、同类服务有几十个副本、调用方凭什么知道该连谁？

ANP 的答案是引入第三方角色：**服务发现中心（Registry）**。所有 Agent 上线时先"登记户口"（注册），需要合作时先"查户口"（发现），再按元数据挑最合适的那个（选择）。三个文件递进：

| 文件 | 讲什么 |
|---|---|
| `11_ANPInit.py` | 三件套入门：注册 → 发现 → 组网 |
| `13_ANPLoadBalancing.py` | 用元数据里的 `load` 字段做最小负载均衡器 |
| `12_ANPTaskDistribution.py` | 把 ANP 发现工具交给 LLM，做智能任务调度 |

（编号 12 在 13 之前，但 13 是 12 的机制基础，建议按 11 → 13 → 12 的顺序读。）

---

## 🧅 第二层：`11_ANPInit.py`——注册：一张服务登记表

```python
discovery = ANPDiscovery()

register_service(
    discovery=discovery,
    service_id="nlp_agent_1",
    service_name="NLP处理专家A",
    service_type="nlp",
    capabilities=["text_analysis", "sentiment_analysis", "ner"],
    endpoint="http://localhost:8001",
    metadata={"load": 0.3, "price": 0.01, "version": "1.0.0"}
)
```

（第 4-15 行）一条注册记录的字段设计就是 ANP 的核心数据模型：

| 字段 | 回答的问题 |
|---|---|
| `service_id` | 你是谁（唯一标识） |
| `service_type` | 你是哪一类（发现时按类检索） |
| `capabilities` | 你具体会什么（细粒度能力标签） |
| `endpoint` | 去哪找你（真正干活时还是回到 A2A/MCP 的 URL） |
| `metadata` | 你现在状态如何（负载、价格、版本……任意键值对） |

注意 `endpoint` 的存在说明 ANP **不替代** A2A/MCP：发现完成后，实际调用仍然走那两个协议。ANP 只管"找到你"，不管"和你说话"。

---

## 🧅 第三层：发现与选择（第 29-37 行）

```python
nlp_services = discover_service(discovery, service_type="nlp")
print(f"找到 {len(nlp_services)} 个NLP服务")

# 选择负载最低的服务
best_service = min(nlp_services, key=lambda s: s.metadata.get("load", 1.0))
```

两步分工明确：

1. **发现**——按 `service_type` 查询，返回候选列表。调用方从头到尾没写过任何 URL；
2. **选择**——`min(..., key=...)` 按 metadata 里的 `load` 挑最闲的。选择策略完全由调用方掌控：换成 `price` 就是成本优先，组合加权就是自定义调度算法。

`.get("load", 1.0)` 的默认值也有讲究：**没上报负载的服务视为满载**（1.0），宁可少用不可压垮——保守兜底。

文件最后（第 42-52 行）用 `ANPNetwork` 把注册过的服务加为节点、`connect_nodes` 建立连接、`get_network_stats()` 查看拓扑——从"一张登记表"进到"一张有边的图"，为大规模组网留了接口。

---

## 🧅 第四层：`13_ANPLoadBalancing.py`——十行写出负载均衡器

注册 5 个同类型 `api` 服务（负载随机），然后（第 20-27 行）：

```python
def get_best_server():
    """选择负载最低的服务器"""
    servers = discovery.discover_services(service_type="api")
    if not servers:
        return None
    best = min(servers, key=lambda s: s.metadata.get("load", 1.0))
    return best
```

模拟循环里藏着让均衡真正"动起来"的一行（第 30-35 行）：

```python
for i in range(10):
    server = get_best_server()
    print(f"请求 {i+1} -> {server.service_name} (负载: {server.metadata['load']:.2f})")
    server.metadata["load"] += 0.1   # 更新负载（模拟）
```

每分配一次请求就把该节点负载 +0.1，于是下一轮 `min` 会自然地选到别人——运行输出里能看到请求像水流一样漫向低洼处，最终各节点负载趋平。**负载均衡不需要专门的中间件，只需要"及时更新的元数据 + 一个 min 函数"**——这就是把状态放进注册表的威力。

---

## 🧅 洋葱心：`12_ANPTaskDistribution.py` 的"LLM 当调度员"（第 33-54 行）

前两个文件的选择策略都是硬编码的 `min(load)`。但真实任务的需求是多维的——"要 GPU"、"要大内存"、"轻量任务别浪费好机器"，写规则会写到吐。这个文件的解法：**把发现工具交给 LLM，让它读着元数据做决策**。

```python
scheduler = SimpleAgent(
    name="任务调度器",
    llm=llm,
    system_prompt="""你是一个智能任务调度器，负责：
1. 分析任务需求
2. 选择最合适的计算节点
3. 分配任务

选择节点时考虑：负载、CPU核心数、内存、GPU等因素。

使用 service_discovery 工具时，必须提供 action 参数：
- 查看所有节点：{"action": "discover_services", "service_type": "compute"}
- 获取网络统计：{"action": "get_stats"}"""
)

anp_tool = ANPTool(
    name="service_discovery",
    description="服务发现工具，可以查找和选择计算节点",
    discovery=discovery
)
scheduler.add_tool(anp_tool)
```

这几行把三章内容焊在了一起：

- **ANP 侧**：事先注册了 10 个计算节点，metadata 里带 `load`、`cpu_cores`、`memory_gb`、`gpu` 四个维度（第 14-28 行，随机生成模拟真实集群）；
- **工具侧**：`ANPTool(discovery=...)` 把注册中心包装成标准 Tool，和 MCPTool、A2ATool 平起平坐；
- **LLM 侧**：system_prompt 里直接给出了工具调用的 JSON 格式示例——又是"提示词定协议"，这次协议的内容是 ANP 的 action 语法。

之后对三种任务发问（第 79-81 行）：

```python
assign_task("训练一个大型深度学习模型，需要GPU支持")
assign_task("处理大量文本数据，需要高内存")
assign_task("运行轻量级数据分析任务")
```

调度器会先调 `discover_services` 拉回 10 个节点的元数据，再用语言理解能力做多目标权衡：GPU 任务挑 `gpu: True` 且负载低的，高内存任务盯 `memory_gb: 64`，轻量任务反而故意选配置一般的节点省资源——**规则引擎写不完的 if/else，LLM 一段话搞定，且能输出选择理由**。这是"智能调度"里"智能"二字的落点。

---

## 🎯 剥完之后：它在本章的位置

ANP 篇回答的问题是：**智能体网络规模化之后，"该找谁"怎么解决？**答案是注册中心三部曲——注册（登记能力与状态）、发现（按类型检索）、选择（按元数据决策，策略可以是 `min` 也可以是 LLM）。

三个协议至此拼成完整版图：

| 协议 | 解决的互通问题 | 关键原语 |
|---|---|---|
| MCP | 智能体 ↔ 工具 | `list_tools` / `call_tool` |
| A2A | 智能体 ↔ 智能体 | Agent Card / `@skill` / `execute_skill` |
| ANP | 智能体 ↔ 网络 | `register_service` / `discover_service` / metadata |

一个成熟的多智能体系统往往三者同用：ANP 找到合适的 Agent，A2A 与它对话，而它内部再用 MCP 调工具干活。下一篇的天气智能体实战，就是这套基础设施在单个垂直场景里的完整走线。
