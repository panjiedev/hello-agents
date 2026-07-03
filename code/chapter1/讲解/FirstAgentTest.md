# FirstAgentTest.py 剥洋葱式讲解

> 源文件：`code/chapter1/FirstAgentTest.py`（210 行）
> 一句话概括：**不用任何 Agent 框架，用一段系统提示词 + 两个工具函数 + 一个 while 循环，手搓出本书的第一个智能体——一个会查天气、荐景点的 ReAct 旅行助手。**

这是全书的"第一次亲密接触"：它故意把所有零件平铺在一个文件里，让你一眼看穿智能体没有魔法。剥洋葱的顺序就按文件里零件的组装顺序来。

```
第一层  整体：一次任务是怎么跑完的
第二层  AGENT_SYSTEM_PROMPT       ——智能体的"宪法"
第三层  两个工具函数               ——智能体的"手"
第四层  OpenAICompatibleClient    ——智能体的"脑"
洋葱心  主循环与正则解析           ——把一切缝合起来的"脊椎"
```

---

## 🧅 第一层：整体——一次任务是怎么跑完的？

用户问："请帮我查询一下今天秦皇岛的天气，然后根据天气推荐一个合适的旅游景点。"运行后你会看到类似的剧本：

```
循环1  模型: Thought: 我需要先查天气
           Action: get_weather(city="秦皇岛")
      程序: Observation: 秦皇岛当前天气：晴，气温25摄氏度
循环2  模型: Thought: 天气晴朗，适合户外，我来搜景点
           Action: get_attraction(city="秦皇岛", weather="晴")
      程序: Observation: 根据搜索，推荐山海关、鸽子窝公园…
循环3  模型: Thought: 信息足够了
           Action: Finish[今天秦皇岛晴，25℃，推荐去山海关…]
```

**要点：模型和程序在打乒乓球。** 模型每次只说一对 Thought + Action（它自己不能执行任何东西）；程序把 Action 当真、真的去调函数，再把结果作为 Observation 拍回去。球来回几次，任务完成。这个模式叫 **ReAct**（Reasoning + Acting），chapter4 会给出它的工程化版本。

---

## 🧅 第二层：`AGENT_SYSTEM_PROMPT`——智能体的"宪法"（第 1-24 行）

```
你是一个智能旅行助手。…
# 可用工具:
- `get_weather(city: str)`: 查询指定城市的实时天气。
- `get_attraction(city: str, weather: str)`: …

# 输出格式要求:
Thought: [你的思考过程和下一步计划]
Action: [你要执行的具体行动]

Action的格式必须是以下之一：
1. 调用工具：function_name(arg_name="arg_value")
2. 结束任务：Finish[最终答案]
```

这段文本干了三件事，缺一不可：

1. **设定身份**（旅行助手）——约束模型的行为范围；
2. **公布工具清单**——注意工具描述写得像函数签名，因为模型将要"伪装成代码调用"的样子来指名工具；
3. **规定输出协议**——Thought/Action 的格式不是给人看的，是**给后面的正则解析器看的**。

"重要提示"里的每一条（每次只输出一对、Action 不换行、必须用 Finish 结束）都对应主循环里的一个解析坑——这份提示词是被真实的解析失败一点点"打补丁"打出来的。**提示词定协议，代码做解析**，本书从第一个文件起就在贯彻这条铁律。

---

## 🧅 第三层：两个工具函数——真实世界的两扇窗（第 27-110 行）

### `get_weather`（第 29-57 行）：调用免费天气 API

```python
url = f"https://wttr.in/{city}?format=j1"
response = requests.get(url)
data = response.json()
```

wttr.in 是个免注册的天气服务，`format=j1` 要 JSON 格式。值得学的是**收尾**：从 JSON 里挖出天气和气温后，函数返回的不是原始数据，而是一句自然语言——

```python
return f"{city}当前天气：{weather_desc}，气温{temp_c}摄氏度"
```

**因为工具的返回值是给 LLM 读的**，一句干净的中文远胜一坨 JSON。

异常处理分了两类：网络错误（`RequestException`）和数据解析错误（`KeyError/IndexError`，比如城市名无效），各自返回一条**描述错误的字符串**而不是抛异常——错误信息会作为 Observation 喂回模型，让它自己看到"城市名可能写错了"并调整。

### `get_attraction`（第 65-103 行）：Tavily 搜索

```python
query = f"'{city}' 在'{weather}'天气下最值得去的旅游景点推荐及理由"
response = tavily.search(query=query, search_depth="basic", include_answer=True)
```

两个细节：查询语句是**用工具参数拼出来的完整问题**（把天气条件也编进去，搜索结果才有针对性）；`include_answer=True` 让 Tavily 直接返回一段综合性回答，省去自己汇总网页摘要——同样是"为 LLM 降噪"。

### 工具注册（第 107-110 行）

```python
available_tools = {"get_weather": get_weather, "get_attraction": get_attraction}
```

一个普通字典：字符串名字 → 真函数。模型说出名字，程序查表执行——chapter4 的 `ToolExecutor` 就是这个字典加上描述信息的封装版。

---

## 🧅 第四层：`OpenAICompatibleClient`——智能体的"脑"（第 112-140 行）

```python
def generate(self, prompt: str, system_prompt: str) -> str:
    messages = [
        {'role': 'system', 'content': system_prompt},
        {'role': 'user', 'content': prompt}
    ]
    response = self.client.chat.completions.create(model=..., messages=messages, stream=False)
    return response.choices[0].message.content
```

薄薄一层封装，就两个看点：

- **双通道输入**：`system` 放宪法（每次不变），`user` 放"到目前为止的剧本"（每次变长）。角色分离让模型清楚哪些是规则、哪些是进展。
- **密钥从 `.env` 读取**（第 144-150 行）：`load_dotenv()` 把项目根目录 `.env` 里的 `OPENAI_API_KEY` 等变量载入环境，代码里不再出现明文密钥。`TAVILY_API_KEY` 也随之进入环境变量，`get_attraction` 里 `os.environ.get` 就能拿到。

chapter4 的 `HelloAgentsLLM` 是它的升级版（流式输出 + 配置瀑布），对比着看能感受到"能跑"和"好用"的差距。

---

## 🧅 洋葱心：主循环与正则解析（第 164-210 行）

一切零件在这 45 行里缝合。骨架：

```python
prompt_history = [f"用户请求: {user_prompt}"]
for i in range(5):                                   # ① 最多5轮，防死循环
    full_prompt = "\n".join(prompt_history)          # ② 剧本重演
    llm_output = llm.generate(full_prompt, AGENT_SYSTEM_PROMPT)
    ...解析 Action...
    observation = available_tools[tool_name](**kwargs)  # ③ 当真执行
    prompt_history.append(f"Observation: {observation}")# ④ 写回剧本
```

**② 是理解智能体"记忆"的钥匙**：LLM 无状态，每一轮都把从头到尾的完整剧本（用户请求 + 历次 Thought/Action/Observation）重新喂一遍。所谓"智能体记得上一步"，就是这个不断变长的列表。

再剥开三处正则——文本与程序之间的海关：

### ① 截断多余输出（第 174-179 行）

```python
match = re.search(r'(Thought:.*?Action:.*?)(?=\n\s*(?:Thought:|Action:|Observation:)|\Z)', llm_output, re.DOTALL)
```

模型有个经典毛病：一口气把后面几轮的 Thought-Action 甚至编造的 Observation 全"抢答"出来。这条正则只保留**第一对** Thought-Action：非贪婪的 `.*?` 配合先行断言 `(?=...)`，取到"下一个 Thought:/Action:/Observation: 标记或文本末尾"为止。抢答的部分直接丢弃——因为 Observation 必须来自真实工具执行，不能让模型自导自演。

### ② 分派 Finish（第 193-196 行）

```python
if action_str.startswith("Finish"):
    final_answer = re.match(r"Finish\[(.*)\]", action_str).group(1)
```

`Finish[...]` 是协议里的停机信号，抓到就取出方括号里的最终答案、`break` 出循环。

### ③ 解析工具调用（第 198-203 行）

```python
tool_name = re.search(r"(\w+)\(", action_str).group(1)      # get_weather
args_str  = re.search(r"\((.*)\)", action_str).group(1)      # city="秦皇岛"
kwargs    = dict(re.findall(r'(\w+)="([^"]*)"', args_str))   # {"city": "秦皇岛"}
observation = available_tools[tool_name](**kwargs)
```

最妙的是第三行：`re.findall` 抓出所有 `键="值"` 对，`dict()` 转成字典，再用 `**kwargs` 解包成关键字参数——**模型输出的一行"伪代码"，就这样变成了一次真实的函数调用**。这三行是"LLM 调用工具"这个魔法的全部真相。

还有一个容易剥漏的防御（第 184-190 行）：解析不到 Action 时，不崩溃，而是把 `"错误: 未能解析到 Action 字段…"` 作为 Observation 写回历史，`continue` 让模型下一轮自己改正——**错误也是反馈**。

---

## 🎯 剥完之后：这个文件教会你的事

把洋葱装回去，智能体 = **提示词（协议） + LLM（决策） + 工具（执行） + 循环（记忆与调度）**。四个零件全在明面上，没有任何框架黑盒。

它的每一处"简陋"，都在后续章节被逐一升级：

| 本文件的做法 | 升级版 |
|---|---|
| 工具字典 `available_tools` | chapter4 `ToolExecutor`（注册表 + 给模型看的描述） |
| `OpenAICompatibleClient` | chapter4 `HelloAgentsLLM`（流式 + 配置瀑布） |
| 走一步看一步的循环 | chapter4 Plan-and-Solve（先规划）、Reflection（后反思） |
| 靠正则解析伪代码 | 生产中的 Function Calling / JSON mode（协议下沉到模型服务） |
| 模型"底层为什么能思考" | chapter3（N-gram → 词向量 → Transformer → Qwen） |

读懂这 210 行，后面所有章节都是在回答同一个问题：**这个手搓玩具的每个零件，怎么做得更好？**
