# my_simple_agent.py 剥洋葱式讲解

> 源文件：`code/chapter7/my_simple_agent.py`（252 行）
> 一句话概括：**继承框架的 `SimpleAgent`，给纯对话 Agent 装上一套自创的轻量工具调用协议 `[TOOL_CALL:工具名:参数]`——从"只会聊天"升级为"边聊边干活"。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

框架自带的 `SimpleAgent` 是最朴素的对话 Agent：收消息、调 LLM、存历史。本文件继承它，加装三样新能力：

1. **工具调用**——LLM 在回答中输出 `[TOOL_CALL:calculator:15*8+32]`，Agent 解析、执行、把结果喂回去；
2. **流式输出**——`stream_run` 逐字吐回答；
3. **动态工具管理**——`add_tool` / `remove_tool` / `list_tools` 随时增删工具。

对比第四章的 ReAct：那是一套重型的 Thought → Action → Observation 循环，每步一次完整 LLM 调用；本文件的协议轻得多——**平时正常聊天，需要工具时才在回答里"夹带"一个标记**，聊天和干活无缝混合。

---

## 🧅 第二层：用提示词"教会" LLM 协议（第 57-79 行）

工具调用能力的一半根本不在代码里，在 `_get_enhanced_system_prompt` 拼出的系统提示词里：

```python
tools_section += "\n## 工具调用格式\n"
tools_section += "当需要使用工具时，请使用以下格式：\n"
tools_section += "`[TOOL_CALL:{tool_name}:{parameters}]`\n"
tools_section += "例如：`[TOOL_CALL:search:Python编程]` 或 `[TOOL_CALL:memory:recall=用户信息]`\n\n"
```

三步走：先列出注册表里所有工具的描述（`get_tools_description()`，这就是工具注册时 `description` 参数的用武之地），再宣布调用格式，最后给出示例（few-shot）。**提示词定协议，代码做解析**——第四章讲解里反复出现的那条工程铁律，在这里长出了第四种形态。

若没配工具注册表，方法直接返回原始提示词（第 61-62 行），Agent 退化为纯聊天——**能力可选，且退化优雅**。

---

## 🧅 第三层：协议的另一半——正则解析（第 130-143 行）

```python
pattern = r'\[TOOL_CALL:([^:]+):([^\]]+)\]'
matches = re.findall(pattern, text)
```

LLM 答应了用这个格式，代码就得能认出这个格式。一条正则、两个捕获组（工具名、参数），`findall` 一次抓出回答中**所有**工具调用。每个匹配还保存了 `original` 原文（第 140 行），后面要用它把标记从回答里抹掉——用户不应该看到协议残渣。

`_parse_tool_parameters`（第 168-194 行）再对参数做一层"智能翻译"：`key=value,key2=value2` 拆成字典；裸字符串则按工具类型猜语义（给 `search` 就当 `query`，给 `memory` 就当搜索词）。这是对 LLM 输出格式不稳定的**宽容设计**：格式协议定得严，解析代码放得宽。

---

## 🧅 洋葱心：多轮工具循环（第 86-117 行）

```python
while current_iteration < max_tool_iterations:
    response = self.llm.invoke(messages, **kwargs)
    tool_calls = self._parse_tool_calls(response)

    if tool_calls:
        for call in tool_calls:
            result = self._execute_tool_call(call['tool_name'], call['parameters'])
            tool_results.append(result)
            clean_response = clean_response.replace(call['original'], "")

        messages.append({"role": "assistant", "content": clean_response})
        messages.append({"role": "user", "content": f"工具执行结果：\n{tool_results_text}\n\n请基于这些结果给出完整的回答。"})
        current_iteration += 1
        continue

    final_response = response   # 没有工具调用 = 最终回答
    break
```

这就是本文件版本的"智能体循环"：

1. 调 LLM → 2. 回答里有工具标记吗？→ 3. 有：执行工具，把**清洗后的回答**和**工具结果**追加进消息列表，回到第 1 步；没有：这就是最终答案，收工。

三个细节值得咀嚼：

- **工具结果以 `user` 角色喂回**（第 110 行）——OpenAI 消息协议里没有 `tool` 之外的标准角色可复用时，把 Observation 伪装成用户发言是最简通用的做法；
- **`max_tool_iterations=3` 是保险丝**——防止 LLM 陷入"调工具→再调工具"的死循环，烧钱又不出活；
- **`clean_response` 抹掉标记再入历史**——协议是 Agent 和 LLM 之间的私房话，不能泄露给用户和后续对话。

---

## 🧅 第五层：流式与工具管理——锦上添花（第 196-252 行）

`stream_run` 用 `self.llm.stream_invoke` 逐 chunk `yield`，边打印边转发，结束后把**完整拼接的回答**存进历史——流式只是传输方式，历史记录里存的永远是完整对话。

`add_tool`（第 227-235 行）有个贴心设计：如果 Agent 建时没配注册表，第一次 `add_tool` 会现场 `ToolRegistry()` 一个并顺手打开 `enable_tool_calling` 开关——**用着用着升级，不用推倒重建**。

---

## 🎯 剥完之后：它在本章的位置

`MySimpleAgent` 是"继承框架基类、重写 `run`"这条扩展路线的第一个完整示范：

| 继承自框架的 | 自己写的 |
|---|---|
| 消息历史（`_history` / `add_message`） | `[TOOL_CALL:...]` 协议的定义与解析 |
| LLM 调用（`invoke` / `stream_invoke`） | 多轮工具循环 `_run_with_tools` |
| `Message` / `Config` / `ToolRegistry` 基建 | 动态工具管理便利方法 |

它和 `MyReActAgent` 是同一枚硬币的两面：SimpleAgent 的工具调用是**嵌入式**的（标记藏在聊天里，轻、快、口语化），ReAct 是**结构式**的（每步强制 Thought/Action，重、稳、可审计）。简单查询用前者，多步推理用后者——`test_simple_agent.py` 会把这里的每个能力挨个点亮。
