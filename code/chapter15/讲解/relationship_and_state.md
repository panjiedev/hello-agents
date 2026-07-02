# 关系与状态 剥洋葱式讲解

> 源文件：`code/chapter15/Helloagents-AI-Town/backend/relationship_manager.py`（324 行）、`state_manager.py`（134 行）、`batch_generator.py`（211 行）
> 一句话概括：**三个文件回答两个问题——NPC 怎么对你"日久生情"（好感度闭环），以及没人搭理时 NPC 怎么"过自己的日子"（批量生成 + 定时刷新）。**

---

## 🧅 第一层（最外层）：这三个文件在干什么？

`agents.py` 让 NPC 能记住对话，但一个活的小镇还差两口气：

1. **关系会演化**——你天天夸张三，他该从"陌生"变"挚友"；你出言不逊，他该冷脸相待。`relationship_manager.py` 负责这个。
2. **世界自己会动**——玩家不搭话时，NPC 也该自言自语、有自己的工作状态。`batch_generator.py` 负责生成这些"自主对话"，`state_manager.py` 负责每 30 秒定时刷新并缓存。

前者是**对话驱动**的（每聊一轮更新一次），后者是**时间驱动**的（不管有没有人玩，后台自己转）——一动一静，撑起小镇的生活感。

---

## 🧅 第二层：用 LLM 当"情感裁判"（relationship_manager.py 第 37-103 行）

好感度怎么算？传统游戏用关键词表（说"谢谢"+2，说"傻"−5），但玩家的话千变万化，词表永远不够用。本项目的做法是：**再雇一个 Agent 专门当裁判**。

```python
self.analyzer_agent = SimpleAgent(
    name="AffinityAnalyzer",
    llm=llm,
    system_prompt=self._create_analyzer_prompt()
)
```

裁判的系统提示词（第 45-103 行）是一份精心设计的"判分细则"：

```
【好感度变化规则】
- 赞美、感谢、请教: +3 到 +8
- 友好问候、正常交流: +1 到 +3
- 普通闲聊、中性话题: 0
- 批评、质疑、不耐烦: -3 到 -8
- 侮辱、攻击、恶意: -8 到 -15

【输出格式】(严格遵守JSON格式,不要添加任何其他文字)
{"should_change": true/false, "change_amount": ..., "reason": "...", "sentiment": "..."}
```

三个提示词工程细节：**量表锚定**（给出明确的分值区间，防止 LLM 随心情打分）、**五个 few-shot 示例**（第 73-96 行，覆盖赞美/批评/中性三类）、**只输出 JSON 的三重强调**（开头、格式段、结尾【重要】各说一遍）。注意负分区间（−15）比正分区间（+10）更大——**关系毁起来比建起来快**，这个不对称是刻意的。

---

## 🧅 第三层：`analyze_and_update_affinity`——判分并执行（第 138-213 行）

每轮对话结束后，`agents.py` 调用这个方法：

```python
response = self.analyzer_agent.run(prompt)     # 裁判看对话
analysis = self._parse_analysis(response)      # 解析判决书

if analysis["should_change"]:
    new_affinity = current_affinity + analysis["change_amount"]
    new_affinity = max(0.0, min(100.0, new_affinity))  # 钳在0-100
    self.set_affinity(npc_name, new_affinity, player_id)
```

配套的 `_parse_analysis`（第 215-264 行）是全项目最典型的**三级降级解析**：

1. 直接 `json.loads`——LLM 听话时一步到位；
2. 失败则找第一个 `{` 和最后一个 `}` 截取再解析——应对 LLM 在 JSON 前后加废话；
3. 再失败就用正则逐字段抠 `"should_change"`、`"change_amount"`——应对 JSON 本身残缺；
4. 全失败返回"不变化"的默认值——**宁可这轮白判，也不让服务崩掉**。

这和 chapter4 讲的"提示词定协议，代码做解析"是同一条铁律，只是这里把"解析"做成了防弹衣。

好感度到人话的两张映射表（第 266-304 行）是闭环的最后一环：`get_affinity_level` 把分数切成五档（陌生→熟悉→友好→亲密→挚友），`get_affinity_modifier` 把分数翻译成对话风格指令（"非常热情友好,像老朋友一样" / "冷淡疏离,回答简短"）——后者会被 `agents.py` 注入下一轮的提示词。

---

## 🧅 第四层：`batch_generator.py`——一次调用养活三个 NPC（第 61-136 行）

NPC 的自主对话（头顶气泡）如果每人每 30 秒调一次 LLM，成本是 6 次/分钟。批量生成器的思路：**把三个人的戏写进一个提示词，一次调用全生成**：

```python
prompt = f"""请为Datawhale办公室的3个NPC生成当前的对话或行为描述。
【场景】{context}
【NPC信息】
- 张三(Python工程师): 在工位区写代码,性格技术宅,喜欢讨论算法和框架
- 李四(产品经理): 在会议室整理需求,...
【输出格式】(严格遵守)
{{"张三": "...", "李四": "...", "王五": "..."}}
"""
```

调用次数从 360 次/小时降到 120 次/小时，**成本直降 66%**。这是多智能体系统落地时最实用的一课：不是每个智能体都值得一次独立调用，"群演"可以合并拍摄。

两个配角设计同样值得剥：

- **`_get_current_context`**（第 170-185 行）：按真实时钟把一天切成六段（清晨/上午/午餐/下午/傍晚/夜晚），场景描述随现实时间变化——玩家中午打开游戏，NPC 真的在"休息放松"；
- **预设对话库**（第 38-59 行）：LLM 不可用时按时段返回手写台词，`generate_batch_dialogues` 的每条失败路径（未启用/解析失败/异常）都落到 `_get_preset_dialogues()`——**没有 API key，小镇也能转**，只是 NPC 变成了"背台词的群演"。

---

## 🧅 第五层：`state_manager.py`——异步心跳与缓存（第 37-114 行）

批量生成器只管"生成一次"，谁来周期性地喊它？状态管理器用 asyncio 起了一个后台任务：

```python
async def start(self):
    await self._update_npc_states()                          # 启动先跑一次
    self._update_task = asyncio.create_task(self._auto_update_loop())

async def _auto_update_loop(self):
    while self._running:
        await asyncio.sleep(self.update_interval)            # 睡30秒
        await self._update_npc_states()                      # 刷新缓存
```

关键的架构选择是**生成与查询彻底解耦**：后台循环把结果写进 `self.current_dialogues` 字典；Godot 轮询 `/npcs/status` 时，`get_current_state()`（第 101-114 行）只是读缓存并算一个倒计时，**零 LLM 调用、毫秒级返回**。一百个玩家同时在线，LLM 调用量也不变。

循环里的异常处理（第 76-78 行）只打印不中断——一次生成失败，30 秒后还有下一次；`asyncio.CancelledError` 单独捕获用于优雅停机（`main.py` 的 lifespan 关闭时调 `stop()`）。

---

## 🧅 洋葱心：好感度的闭环（relationship_manager.py 第 172-178 行 ✕ agents.py 第 194-198 行）

```python
# 本轮对话结束: 裁判判分,更新分数
new_affinity = current_affinity + analysis["change_amount"]
self.set_affinity(npc_name, new_affinity, player_id)

# 下一轮对话开始: 分数翻译成风格,注入提示词 (agents.py)
affinity_context = f"""【当前关系】
你与玩家的关系: {affinity_level} (好感度: {affinity:.0f}/100)
【对话风格】{affinity_modifier}
"""
```

把这两段拼起来看，才能看到本篇真正的心脏——一个**跨越对话轮次的反馈回路**：

```
你说话 → NPC回复 → 裁判LLM判分 → 好感度变化 → 翻译成风格指令
   ↑                                                    ↓
   └────────────── 下一轮NPC的语气变了 ←─────────────────┘
```

LLM 在回路里出现了两次，扮演两个角色：一次是"演员"（生成回复），一次是"裁判"(分析情感）。数字（0-100）负责**积累**，自然语言负责**表达**，两者通过 `get_affinity_modifier` 这张映射表互相转换。你对张三的每句话都在悄悄改写他下一次开口的语气——这就是"关系演化"的全部实现，总共不过几十行代码。

---

## 🎯 剥完之后：它在本章的位置

三个文件补齐了 AI 小镇的"社会性"和"生活感"：

| 文件 | 驱动方式 | LLM 角色 | 输出去向 |
|---|---|---|---|
| `relationship_manager.py` | 对话驱动（每轮一次） | 裁判：情感分析 | 好感度 → 下轮提示词 + `/affinities` 接口 |
| `batch_generator.py` | 被动调用 | 编剧：批量写台词 | 对话字典 → 状态缓存 |
| `state_manager.py` | 时间驱动（30 秒心跳） | 不碰 LLM | 缓存 → `/npcs/status` → Godot 气泡 |

和 `agents.py` 合起来，后端形成了两条互不干扰的通路：**即时对话通路**（玩家按 E → `/chat` → 记忆+好感度流水线）和**自主行为通路**（定时器 → 批量生成 → 缓存 → 轮询）。前者贵而深，后者廉而广——这种"分级供给智能"的思路，是把 LLM 塞进实时游戏的关键取舍。
