# 记忆工具基础（01-03）剥洋葱式讲解

> 源文件：
> - `code/chapter8/01_MemoryTool_Basic_Operations.py`（198 行）
> - `code/chapter8/02_MemoryTool_Architecture.py`（247 行）
> - `code/chapter8/03_WorkingMemory_Implementation.py`（303 行）
>
> 一句话概括：**给智能体装上"记性"——MemoryTool 用一个统一的 `run({"action": ...})` 接口管理四种记忆（工作 / 情景 / 语义 / 感知），01 演示怎么用，02 剖开架构，03 深挖工作记忆的实现机制。**

---

## 🧅 第一层（最外层）：这三个文件在干什么？

前几章的智能体都是"金鱼"：每次对话从零开始，上一轮学到的东西一概不记得。本章要治这个病，第一味药就是 `MemoryTool`。

三个文件是同一件事的三个视角：

| 文件 | 视角 | 回答的问题 |
|---|---|---|
| 01 | 用户视角 | 记忆工具怎么用？（增、查、忘、整合） |
| 02 | 架构师视角 | 它内部是怎么组织的？（分层 + 组合模式） |
| 03 | 实现者视角 | 最常用的工作记忆是怎么实现的？（容量、TTL、混合检索） |

三个文件调用的始终是同一个入口：

```python
memory_tool.run({"action": "...", ...})   # 一个字典进去，一个字符串结果出来
```

---

## 🧅 第二层：四种记忆的"人格分裂"（01 第 36-76 行）

01 号文件开场就把四种记忆各添加了一条，四段代码结构完全相同，只有 `memory_type` 和元数据不同：

```python
memory_tool.run({
    "action":"add",
    "content":"正在学习HelloAgents框架的记忆系统",
    "memory_type":"working",          # ← 换这个
    "importance":0.7,
    "task_type":"learning"            # ← 和这些专属元数据
})
```

对照四次调用（第 36-76 行），能读出每种记忆的"性格"：

| 记忆类型 | 专属元数据 | 对应人类认知 |
|---|---|---|
| `working` 工作记忆 | `task_type` | 脑子里正在转的事，转瞬即逝 |
| `episodic` 情景记忆 | `event_type`、`location` | "2024 年在研发中心开始研究 Agent"——**发生过什么** |
| `semantic` 语义记忆 | `concept`、`domain` | "记忆系统有四种类型"——**知道什么** |
| `perceptual` 感知记忆 | `modality`、`source` | "看过架构图"——**感知过什么** |

注意 `importance` 的取值也有讲究：语义知识 0.9 > 人生里程碑 0.8 > 手头任务 0.7 > 随手一瞥 0.6——重要性不是装饰，后面的检索排序、遗忘、整合全靠它。

搜索同样走 `run`（第 85-105 行），支持三种过滤：普通语义搜索、按 `memory_type` 圈定范围、按 `min_importance` 设置门槛。

而记忆管理的两个"狠角色"在第 138-152 行登场：

```python
memory_tool.run({"action":"forget", "strategy":"importance_based", "threshold":0.2})
memory_tool.run({"action":"consolidate", "from_type":"working",
                 "to_type":"episodic", "importance_threshold":0.6})
```

`forget` 把不重要的忘掉，`consolidate` 把重要的升级为长期记忆——这两个动作是第 06 篇（记忆巩固）的主角，这里先混个脸熟。

---

## 🧅 第三层：剖开架构——三层洋葱套娃（02 第 34-68 行）

02 号文件回答"这个工具内部长什么样"。初始化时传三样东西：

```python
memory_tool = MemoryTool(
    user_id="architecture_demo_user",     # 谁的记忆（多用户隔离）
    memory_config=self.memory_config,     # 怎么配置
    memory_types=self.memory_types        # 启用哪几种记忆
)
```

第 58-68 行顺着属性往里挖，挖出了三层结构：

```python
memory_manager = memory_tool.memory_manager          # 第二层：管理器
for memory_type, memory_instance in memory_manager.memory_types.items():
    print(f"  • {memory_type}: {type(memory_instance).__name__}")   # 第三层：具体组件
```

```
MemoryTool（工具外壳，对接 Agent 的工具协议）
  └─ MemoryManager（调度中枢，统一路由）
       ├─ WorkingMemory     （纯内存，60 分钟 TTL）
       ├─ EpisodicMemory    （SQLite + Qdrant 混合存储）
       ├─ SemanticMemory    （Neo4j 知识图谱 + Qdrant 向量）
       └─ PerceptualMemory  （分模态向量存储）
```

这是教科书级的**组合模式**：`MemoryManager` 持有一个 `{类型名: 记忆实例}` 字典，每种记忆独立实现、独立存储，`add`/`search` 请求按 `memory_type` 路由到对应组件。第 75-100 行的表格式注释点明了各组件的存储选型——工作记忆图快（纯内存），情景记忆要顺序（SQLite），语义记忆要关系（Neo4j），感知记忆要跨模态（分模态向量库）。

扩展性落在两处（第 182-195 行）：

```python
custom_config = MemoryConfig()
custom_config.working_memory_capacity = 100        # 改容量
custom_config.working_memory_ttl_minutes = 120     # 改保鲜期

selective_memory_tool = MemoryTool(..., memory_types=["working", "semantic"])  # 只开两种
```

不用的记忆类型不初始化——用不到 Neo4j 就别连 Neo4j，这在实际部署时很实惠。

---

## 🧅 第四层：工作记忆的四板斧（03）

03 号文件盯着最高频的工作记忆，剥出四个机制：

**① 容量 + 优先级（第 38-47 行）**：默认只装 50 条，塞满后按重要性排挤。演示代码故意用 `importance = 0.3 + (i * 0.07)` 添加递增重要性的记忆，让你亲眼看到搜索结果按重要性排序。

**② 混合检索（第 68-72 行）**：一次搜索背后是四个信号的加权融合——

```
最终得分 = TF-IDF 语义相似度 + 关键词匹配 + 时间衰减因子 + 重要性权重
```

第 115-131 行用四组查询分别戳这四个信号："Python编程"测语义、"学习"测关键词、"复杂度"测部分匹配、"人工智能机器学习"测多词。

**③ 时间衰减（第 145-172 行）**：同样 `importance=0.7` 的四条记忆，新的排在旧的前面——"刚学的"天然比"上周学的"更该被想起来，这是对人类记忆遗忘曲线的直接模仿。

**④ 自动清理（第 203-207 行）**：TTL 到期自动清 + 手动 `forget` 按阈值清，保证这块"内存"永远轻快。第 229-251 行的性能测试给出理由：纯内存存储，20 次添加、10 次搜索都在亚秒级——工作记忆就该像 CPU 缓存一样快。

---

## 🧅 洋葱心：一个字典协议统治所有操作

三个文件剥到最后，核心就是这一个调用形状（01 第 36-42 行是它的第一次亮相）：

```python
result = memory_tool.run({
    "action": "add",          # 动词：add / search / forget / consolidate / stats ...
    "content": "...",         # 宾语
    "memory_type": "working", # 路由键：决定进哪个记忆组件
    "importance": 0.7,        # 生命线：决定排序、遗忘、整合的命运
    **任意元数据               # 自由扩展：task_type / location / concept / modality ...
})
```

为什么是字典而不是一堆方法（`add_memory()`、`search_memory()`……）？因为**这个接口最终是给 LLM 用的**：第八篇（Agent 集成）里，Agent 生成的工具调用就是一段 JSON——字典协议让"LLM 的输出"和"工具的输入"天然同构，中间不需要任何翻译层。

字典里的字段还形成了清晰的责任分工：`action` 是路由、`memory_type` 是二级路由、`importance` 是全生命周期的调度依据、其余键全部作为元数据透传存储。**一个入口，无限扩展**——这是本章所有工具（包括 RAGTool）共守的设计契约。

---

## 🎯 剥完之后：它在本章的位置

`MemoryTool` 是本章"记忆"这条线的地基：

| 后续文件 | 在这个地基上盖什么 |
|---|---|
| 06 记忆巩固 | 把 `consolidate` 拆开细看：短期记忆如何升级为长期记忆 |
| 09 记忆类型深挖 | 把四种记忆各自单独拎出来做深度体检 |
| 08 Agent 集成 | 把 MemoryTool 注册进 ToolRegistry，交给 Agent 自主调用 |
| 11 问答助手 | 用它记录用户的学习历程，实现"我之前学过什么？" |

和下一篇 RAG 管线对照着看会更有味道：**RAG 记的是"世界的知识"（外部文档），Memory 记的是"自己的经历"（交互历史）**——一个智能体想要真正"有记性"，两者缺一不可。
