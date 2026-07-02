# AgentScopeDemo（三国狼人杀）剥洋葱式讲解

> 源文件：`code/chapter6/AgentScopeDemo/`（五个文件：`game_roles.py` 114 行、`prompt_cn.py` 57 行、`structured_output_cn.py` 138 行、`utils_cn.py` 172 行、`main_cn.py` 383 行）
> 一句话概括：**用 AgentScope 搭一局"三国狼人杀"：九位三国人物由 `ReActAgent` 扮演，靠 `MsgHub` 消息中心分组"睁眼闭眼"，靠 pipeline 编排发言与投票，靠 Pydantic 结构化输出保证每次行动都能被程序解析。**

---

## 🧅 第一层（最外层）：这个 Demo 在干什么？

这是本章规模最大的例子——一个完整的多智能体**博弈**：狼人夜里密谋杀人，预言家验人，女巫用药，白天全员讨论投票……它对框架提出了前三个 Demo 都没有的三个硬要求：

1. **信息隔离**：狼人讨论只有狼人听得见，白天发言全员可见——通信拓扑要能动态分组；
2. **动作可解析**：投票、验人、开枪的结果必须是程序能读的数据，不能是自由文本；
3. **复杂流程**：夜晚五个阶段 + 白天两个阶段 + 胜负判定，循环最多十轮。

五个文件恰好各管一层：`game_roles.py` 管规则数据，`prompt_cn.py` 管人设，`structured_output_cn.py` 管输出格式，`utils_cn.py` 管主持人和裁判，`main_cn.py` 管全场编排。下面逐个剥。

---

## 🧅 第二层：`game_roles.py` + `prompt_cn.py`——规则与人设（纯数据层）

`game_roles.py` 里没有一行 AI 代码，只有两张字典表和几个查询方法：

- `ROLES`（第 9-46 行）：六种职业的技能与胜利条件；
- `CHARACTER_TRAITS`（第 48-58 行）：九位三国人物的说话风格——"刘备：仁德宽厚……张飞：说话大声直接，容易冲动"；
- `get_standard_setup()`（第 86-114 行）：按人数返回职业配置（6 人局 = 2 狼 + 预言家 + 女巫 + 2 村民），人数不标准时用"约 1/3 狼人"的公式兜底。

`prompt_cn.py` 只做一件事：把"职业规则 + 三国人设"拼成系统提示词。`get_role_prompt(role, character)`（第 7-56 行）先给一段公共开头，再按职业追加分工说明，例如狼人版：

```python
- 你是狼人阵营，目标是消灭所有好人
- 夜晚可以与其他狼人协商击杀目标
- 白天要隐藏身份，误导好人
- 以{character}的性格说话和行动
```

（第 23-28 行）**游戏性完全由提示词注入**：同一个 `ReActAgent` 类，喂"狼人 × 曹操"就阴险狡诈，喂"预言家 × 诸葛亮"就明察秋毫。这与 AutoGen 的"提示词即岗位"是同一思想，只是岗位换成了角色卡。

---

## 🧅 第三层：`structured_output_cn.py`——用 Pydantic 给发言"上笼头"

游戏最怕模型输出"我觉得可能是刘备吧？"这种没法统计的票。本文件为每类动作定义一个 Pydantic 模型，如狼人击杀（第 106-118 行）：

```python
class WerewolfKillModelCN(BaseModel):
    target: str = Field(description="要击杀的玩家姓名")
    kill_strategy: str = Field(description="击杀策略说明")
```

最漂亮的一招是**动态生成的合法选项**（第 24-41 行）：

```python
def get_vote_model_cn(agents: list[AgentBase]) -> type[BaseModel]:
    class VoteModelCN(BaseModel):
        vote: Literal[tuple(_.name for _ in agents)] = Field(
            description="你要投票淘汰的玩家姓名",
        )
        ...
    return VoteModelCN
```

这是一个"模型工厂"：每次投票前，用**当前存活玩家名单**现场生成一个 Pydantic 类，`Literal[("刘备", "曹操", ...)]` 把合法票面钉死在类型系统里——**投给死人或不存在的人，在 schema 层面就非法**。预言家（第 65-82 行）和猎人（第 85-103 行）的目标选择用了同样的工厂套路。

对比 chapter4 用正则从自由文本里抠 `Finish[...]`，这里是"协议即类型"的升级版：把输出协议从提示词约定升格为机器可校验的 JSON Schema。

---

## 🧅 第四层：`utils_cn.py`——不带 LLM 的主持人与铁面裁判

`GameModerator`（第 97-142 行）是个耐人寻味的设计——它继承了 `AgentBase`，却**不接任何模型**：

```python
class GameModerator(AgentBase):
    async def announce(self, content: str) -> Msg:
        msg = Msg(name=self.name, content=f"📢 {content}", role="system")
        self.game_log.append(content)
        await self.print(msg)
        return msg
```

（第 105-114 行）它说明 AgentScope 里"智能体"的本质不是"会调 LLM"，而是"**能收发 `Msg` 消息**"。主持人的公告作为 `role="system"` 的消息进入游戏，返回的 `Msg` 对象随后被喂给玩家的 `observe()` 或 `MsgHub` 的 `announcement`——**消息对象是全场唯一的流通货币**。

两个"铁面"函数则是纯 Python 的确定性逻辑：

- `majority_vote_cn`（第 40-48 行）：`Counter(votes.values()).most_common(1)` 一行定生死——**计票绝不交给 LLM**；
- `check_winning_cn`（第 51-62 行）：数狼人和好人的存活数，判定胜负。

博弈类系统的通用原则在此显形：**LLM 负责表演和决策，规则引擎负责裁决**，二者边界清晰。

---

## 🧅 第五层：`main_cn.py`——AgentScope 的三大编排武器

### 5.1 定义玩家：`ReActAgent`（第 60-69 行）

```python
agent = ReActAgent(
    name=name,
    sys_prompt=ChinesePrompts.get_role_prompt(role, character),
    model=DashScopeChatModel(
        model_name="qwen-max",
        api_key=os.environ["DASHSCOPE_API_KEY"],
        enable_thinking=True,
    ),
    formatter=DashScopeMultiAgentFormatter(),
)
```

四个参数各司其职：名字、人设、脑子、以及一个别家框架少见的 **`formatter`**——多智能体场景下，一个 Agent 收到的历史消息来自许多不同名字的发言者，formatter 负责把这团消息整理成底层模型 API 能接受的格式。紧接着第 72-77 行用 `agent.observe(...)` 私聊告知身份：**`observe` 是"只入耳、不回话"的单向注入**，与要求回复的 `agent(...)` 调用相对。

### 5.2 信息隔离：`MsgHub`（第 125-144 行，狼人夜谈）

```python
async with MsgHub(
    self.werewolves,
    enable_auto_broadcast=True,
    announcement=await self.moderator.announce("狼人们，请讨论今晚的击杀目标。..."),
) as werewolves_hub:
    for _ in range(MAX_DISCUSSION_ROUND):
        for wolf in self.werewolves:
            await wolf(structured_model=DiscussionModelCN)

    werewolves_hub.set_auto_broadcast(False)
```

`MsgHub` 是 AgentScope 的招牌：一个**用 `async with` 圈出来的临时聊天室**。圈内成员（这里只有狼人）的发言自动广播给彼此，圈外人（好人）毫不知情——"天黑请闭眼"的信息隔离，就靠参与者列表实现。白天讨论（第 276-282 行）则换成 `MsgHub(self.alive_players, ...)`，全员入圈。投票前一句 `set_auto_broadcast(False)` 关掉广播，让每张票互相保密——**通信拓扑在运行时随游戏阶段动态开合**。

### 5.3 两种节奏：`sequential_pipeline` 与 `fanout_pipeline`（第 284-293 行）

```python
await sequential_pipeline(self.alive_players)          # 白天轮流发言：后发言者听得到前面的话

vote_msgs = await fanout_pipeline(                     # 投票：并发各投各的，互不影响
    self.alive_players,
    await self.moderator.announce("请投票选择要淘汰的玩家"),
    structured_model=get_vote_model_cn(self.alive_players),
    enable_gather=False,
)
```

- **sequential**：一个接一个，配合 MsgHub 广播，形成"张飞听完刘备再发言"的顺序讨论；
- **fanout**：同一条消息扇出给所有人**并行**作答，天然适合投票——既快，又保证互不干扰。

拿到 `vote_msgs` 后从每条消息的 `metadata` 读结构化结果（第 297-304 行），注意满屏的防御式检查：`if vote_msg is not None and hasattr(vote_msg, 'metadata')...`，解析失败视为弃票或随机——**LLM 的输出永远要留兜底**，chapter4 的老规矩。

### 5.4 主循环（第 311-360 行）

`run_game()` 把一切串成回合制：夜晚（狼杀 → 预言家验 → 女巫救/毒）→ 结算死亡 → `check_winning_cn` 判胜负 → 白天（讨论 → 投票 → 猎人开枪）→ 再判胜负，最多 `MAX_GAME_ROUND = 10` 轮。**阶段顺序是硬编码的 Python 控制流**——AgentScope 不像 AutoGen 把流程交给群聊涌现，而是让开发者用普通代码当"上帝之手"，框架只提供通信（MsgHub）和调度（pipeline）的积木。

---

## 🧅 洋葱心：一次带结构化约束的智能体调用

全场几百行编排，最终都收敛到这一个调用形态（第 170-172 行，预言家验人）：

```python
check_result = await seer_agent(
    structured_model=get_seer_model_cn(self.alive_players)
)
target_name = check_result.metadata.get("target")
```

一行代码里叠了 AgentScope 的三层设计：

1. **`await agent(...)`**——智能体是可调用对象，调用即"轮到你发言"，输入来自它此前 `observe` 到的所有消息；
2. **`structured_model=...`**——传入一个按当前局势现做的 Pydantic 类，强制这次发言附带合法的结构化数据；
3. **`.metadata.get("target")`**——自由发言进 `content` 供人看，结构化结果进 `metadata` 供程序读，**表演与数据双轨分离**。

"谁能听见"交给 MsgHub，"谁先谁后"交给 pipeline，"说什么格式"交给 structured_model——三个正交的旋钮，拧出一整局狼人杀。

---

## 🎯 剥完之后：它在本章的位置

AgentScope 走的是"**消息基础设施**"路线：不预设协作范式，把通信、调度、格式约束做成三种独立积木，复杂交互由开发者显式拼装。

| 框架关键问题 | AgentScope 的回答 |
|---|---|
| Agent 怎么定义？ | `ReActAgent(name, sys_prompt, model, formatter)`；不带 LLM 的 `AgentBase` 也是 Agent |
| 多 Agent 怎么通信？ | `Msg` 对象 + `MsgHub` 动态分组广播 + `observe` 定向注入 |
| 流程谁控制？ | 开发者的 Python 代码 + `sequential/fanout_pipeline` 调度积木 |
| 输出怎么保真？ | 每次调用可挂 Pydantic `structured_model`，结果走 `metadata` |

四个 Demo 里它离"裸写"最近、可控性最强，也最能体现多智能体系统的工程全貌：**提示词造角色，类型系统定协议，消息中心管耳朵，纯代码当裁判。**
