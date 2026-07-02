# agents.py 剥洋葱式讲解

> 源文件：`code/chapter15/Helloagents-AI-Town/backend/agents.py`（483 行）
> 一句话概括：**本章的灵魂文件——给每个 NPC 配一个 SimpleAgent 大脑、一套双层记忆系统和一份好感度档案，让"张三"在第二次见面时还记得你说过什么、对你是什么态度。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

如果只用前面章节的知识做 NPC 对话，一个 `SimpleAgent` 加一段角色提示词就够了。但那样的 NPC 是"金鱼"：关掉对话框，一切归零；你昨天夸过他的代码，今天他照样把你当陌生人。

`agents.py` 要造的是有连续人格的 NPC，为此它给每个 Agent 外挂了两样东西：

- **记忆**（`MemoryManager`）——对话结束后存档，下次对话前检索，SQLite 落盘（`memory_data/张三/memory.db`），重启服务也不丢；
- **关系**（`RelationshipManager`）——0-100 的好感度分数，每轮对话后用另一个 LLM 分析"这轮聊天该加分还是扣分"。

对外的核心接口只有一个：

```python
npc_manager.chat("张三", "你好,你在做什么?")  # 返回张三的回复字符串
```

但这一行背后是一条六步流水线——本篇要剥的就是它。

---

## 🧅 第二层：`NPC_ROLES`——人设即数据（第 21-49 行）

```python
NPC_ROLES = {
    "张三": {
        "title": "Python工程师",
        "personality": "技术宅,喜欢讨论算法和框架",
        "expertise": "多智能体系统、HelloAgents框架、Python开发、代码优化",
        "style": "简洁专业,喜欢用技术术语,偶尔吐槽bug",
        "hobbies": "看技术博客、刷LeetCode、研究新框架"
    },
    ...
}
```

三个 NPC（工程师张三、产品经理李四、设计师王五）的全部人格都是**纯数据**——性格、专长、说话风格、爱好，一个字典搞定。想加第四个 NPC？加一个键即可，不用碰任何逻辑代码。这个字典还被 `batch_generator.py` 复用来生成自主对话，是全后端唯一的"角色真相来源"。

`create_system_prompt`（第 51-84 行）把这份数据渲染成系统提示词，其中的【行为准则】和【重要】段落值得细读：

```python
1. 保持角色一致性,用第一人称"我"回答
2. 回复简洁自然,控制在30-50字以内
...
- 不要说"我是AI"或"我是语言模型"
```

"控制在 30-50 字"不是审美偏好——游戏对话框就那么大，且玩家等不了长篇大论；"不要说我是 AI"则是在提示词层面维护游戏的沉浸感。**游戏场景的约束最终都变成了提示词里的一行字。**

---

## 🧅 第三层：`_create_memory_manager`——给 NPC 装海马体（第 140-168 行）

```python
memory_config = MemoryConfig(
    storage_path=memory_dir,
    working_memory_capacity=10,   # 最近10条对话
    max_capacity=100,             # 最多100条长期记忆
    importance_threshold=0.3,     # 检索时只关注重要性较高的记忆
    decay_factor=0.95             # 时间衰减系数
)
memory_manager = MemoryManager(
    config=memory_config,
    user_id=npc_name,        # 用NPC名字隔离存储
    enable_working=True,     # 工作记忆 (短期)
    enable_episodic=True,    # 情景记忆 (长期)
    enable_semantic=False,
    enable_perceptual=False
)
```

这是对 HelloAgents 框架记忆模块（第八章）的一次实战选型，每个参数都对应人类记忆的一个特性：

- **工作记忆**容量 10 条——像人的短期记忆，只装"刚刚聊了什么"；
- **情景记忆**上限 100 条——长期记忆，重要的对话会从工作记忆整合进来；
- **`decay_factor=0.95`**——记忆随时间衰减，久远的对话检索权重越来越低，NPC 也会"渐渐淡忘"；
- **`user_id=npc_name`**——张三的记忆存 `memory_data/张三/`，三个 NPC 的记忆物理隔离，绝不会张三"记得"只对李四说过的话。

四种记忆类型只开了两种：NPC 不需要语义记忆（知识库）和感知记忆（图像/音频）——**按需裁剪框架能力，也是工程判断的一部分。**

---

## 🧅 第四层：`chat()` 的六步流水线（第 170-261 行）

这是全文件最长也最重要的方法。玩家一句"你好"进来，要走完六步：

```
① 取好感度 → ② 检索记忆 → ③ 拼增强提示词 → ④ LLM生成回复 → ⑤ 分析并更新好感度 → ⑥ 存记忆
```

**① 好感度上下文**（第 187-199 行）：

```python
affinity = self.relationship_manager.get_affinity(npc_name, player_id)
affinity_context = f"""【当前关系】
你与玩家的关系: {affinity_level} (好感度: {affinity:.0f}/100)
【对话风格】{affinity_modifier}
"""
```

好感度不是只给 UI 看的数字——它被翻译成自然语言（"友好热情,愿意多聊" / "冷淡疏离,回答简短"）直接注入提示词，**让 LLM 用语气演出关系的远近**。

**② 记忆检索**（第 201-210 行）：

```python
relevant_memories = memory_manager.retrieve_memories(
    query=message,                          # 用玩家这句话做检索
    memory_types=["working", "episodic"],
    limit=5,
    min_importance=0.3
)
```

以玩家当前消息为 query 做语义检索，短期长期一起查，最多取 5 条、过滤掉不重要的——这就是 RAG 思想在游戏里的落地：**不是把全部历史塞进上下文，而是按需取回最相关的几条。**

**⑤ 好感度分析**（第 225-238 行）：把"玩家消息 + NPC 回复"交给 `RelationshipManager` 里的另一个 Agent 做情感分析，决定加减分（细节见"关系与状态篇"）。

**⑥ 记忆写入**（第 240-250 行）：调用 `_save_conversation_to_memory`，把这轮对话变成 NPC 的新记忆。

注意第 257-261 行的兜底：任何一步炸了，返回的不是堆栈而是"抱歉,我现在有点忙,等会儿再聊吧"——**报错也要在角色里**，这是游戏 NPC 和命令行工具的本质区别。

---

## 🧅 第五层：记忆的读与写——两个私有方法

**读**：`_build_memory_context`（第 263-276 行）把检索到的 `MemoryItem` 渲染成一段带时间戳的文本：

```
【之前的对话记忆】
[14:32] 玩家说: 你的代码写得真棒!
[14:32] 我说: 谢谢!我最近在研究新技术。
```

**写**：`_save_conversation_to_memory`（第 278-334 行）每轮对话存**两条**记忆——玩家的话（重要性 0.5）和自己的回复（重要性 0.6），都先进工作记忆，由框架按重要性阈值决定是否整合进长期记忆。最妙的是 metadata：

```python
metadata={
    "speaker": "player",
    "affinity": affinity,            # ⭐ 记录当时的好感度
    "affinity_change": affinity_change,
    "sentiment": sentiment,          # ⭐ 记录情感倾向
    ...
}
```

记忆里同时封存了**"当时说了什么"和"当时关系如何、感受如何"**——就像人回忆一段对话时，记住的不只是内容，还有当时的心情。视角也讲究：玩家的话存成"玩家说: ..."，自己的话存成"**我**说: ..."——记忆是第一人称的。

---

## 🧅 洋葱心：增强提示词的拼装（第 212-222 行）

```python
memory_context = self._build_memory_context(relevant_memories)

enhanced_message = affinity_context
if memory_context:
    enhanced_message += f"{memory_context}\n\n"
enhanced_message += f"【当前对话】\n玩家: {message}"

response = agent.run(enhanced_message)
```

整个文件 483 行，最核心的就是这几行拼接。玩家发的明明只有一句"你好"，NPC 实际"听到"的却是：

```
【当前关系】你与玩家的关系: 亲密 (好感度: 65/100)
【对话风格】友好热情,愿意多聊,会主动关心对方

【之前的对话记忆】
[14:32] 玩家说: 你的代码写得真棒!
[14:32] 我说: 谢谢!我最近在研究新技术。

【当前对话】
玩家: 你好
```

**Agent 本身是无状态的**（`SimpleAgent` 每次 `run` 都从系统提示词开始），NPC 的"人格连续性"完全靠这段运行时拼装的上下文注入。记忆系统、好感度系统兜了一大圈，最终都汇流到这几行字符串拼接上——这就是生成式智能体（Generative Agents）的核心机制：**状态存在 Agent 之外，按需编织进提示词。**

---

## 🎯 剥完之后：它在本章的位置

`agents.py` 是 AI 小镇的"人物内核"，其余模块都围着它转：

| 协作方 | 关系 |
|---|---|
| `main.py` | `/chat` 路由把玩家消息委托给 `NPCAgentManager.chat()` |
| `relationship_manager.py` | 被 `chat()` 的第①⑤步调用，供给好感度、回收情感分析 |
| `logger.py` | 六步流水线每一步都打点，日志就是流水线的可视化 |
| `batch_generator.py` | 复用 `NPC_ROLES` 生成 NPC 的自主对话（不走记忆流水线） |
| `memory_data/` | 每个 NPC 一个 SQLite 库，NPC 的"人生"落盘于此 |

对比前面章节：chapter4 的智能体解决"如何思考"（ReAct / 反思），chapter8 的记忆模块解决"如何记住"，本文件把两者组装进一个活的角色——**智能体不再是回答问题的工具，而是有过去、有态度、住在某个世界里的"人"。**
