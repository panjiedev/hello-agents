# my_react_agent.py 剥洋葱式讲解

> 源文件：`code/chapter7/my_react_agent.py`（100 行）
> 一句话概括：**第四章手写 ReAct 的"框架化重生"——继承 `ReActAgent` 基类，自带可替换的提示词模板，100 行写完当年两百行才能表达的循环。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

如果你读过第四章的 `ReAct.py`，这个文件会有强烈的既视感：同样的 Thought → Action → Observation 循环、同样的 `Finish[答案]` 终止暗号、同样的 `max_steps` 保险丝。区别在于脚下踩的东西变了：

| | 第四章 `ReAct.py` | 本章 `MyReActAgent` |
|---|---|---|
| LLM 调用 | 自己写 `llm_client.py` | 继承来的 `self.llm.invoke` |
| 工具管理 | 自己写 `ToolExecutor` | 框架的 `ToolRegistry` |
| 输出解析 | 自己写正则 | 继承父类的 `_parse_output` / `_parse_action` |
| 对话历史 | 没有 | 继承来的 `add_message` / `get_history` |

文件从零开始写的只有两样东西：**一份提示词模板 + 一个 `run` 方法**。这正是框架的价值主张：范式的"骨架"沉淀进基类，你只填"血肉"。

---

## 🧅 第二层：提示词模板——先立协议（第 1-27 行）

文件开头 27 行不是代码，是一段带三个插槽的提示词：

```python
MY_REACT_PROMPT = """你是一个具备推理和行动能力的AI助手。...

## 可用工具
{tools}
...
Thought: 你的思考过程，...
Action: 你决定采取的行动，必须是以下格式之一：
- `{{tool_name}}[{{tool_input}}]` - 调用指定工具
- `Finish[最终答案]` - 当你有足够信息给出最终答案时
...
**Question:** {question}

## 执行历史
{history}
"""
```

三个插槽 `{tools}` / `{question}` / `{history}` 每一步循环都会重新填充。注意 `{{tool_name}}` 的**双花括号**：这是 `str.format` 的转义——单花括号是要被替换的插槽，双花括号是要原样展示给 LLM 看的格式示范。写模板时漏转义，`format` 会当场 KeyError，这是新手最常踩的坑。

模板被定义成**模块级常量**而非硬编码在方法里，配合构造参数 `custom_prompt`（第 46、52 行），用户可以整体换掉它——`test_react_agent.py` 里就演示了换成"数学专家版"精简提示词。**提示词即配置，配置就该可注入。**

---

## 🧅 第三层：构造函数——组装三大件（第 38-53 行）

```python
super().__init__(name, llm, system_prompt, config)
self.tool_registry = tool_registry
self.max_steps = max_steps
self.current_history: List[str] = []
self.prompt_template = custom_prompt if custom_prompt else MY_REACT_PROMPT
```

先 `super()` 把名字、LLM、配置交给基类托管，再挂上 ReAct 特有的三样：工具注册表（手）、步数上限（保险丝）、执行历史（草稿纸）。注意有**两本账**：`current_history` 是单次任务内的 Action/Observation 草稿（每次 `run` 清空重来），基类的 `_history` 是跨任务的正式对话记录（只存最终问答）。草稿给 LLM 看，正式账给用户看。

---

## 🧅 洋葱心：五步一循环的 `run`（第 62-94 行）

```python
while current_step < self.max_steps:
    # 1. 构建提示词
    prompt = self.prompt_template.format(tools=tools_desc, question=input_text, history=history_str)
    # 2. 调用LLM
    response_text = self.llm.invoke(messages, **kwargs)
    # 3. 解析输出
    thought, action = self._parse_output(response_text)
    # 4. 检查完成条件
    if action and action.startswith("Finish"):
        final_answer = self._parse_action_input(action)
        ...
        return final_answer
    # 5. 执行工具调用
    if action:
        tool_name, tool_input = self._parse_action(action)
        observation = self.tool_registry.execute_tool(tool_name, tool_input)
        self.current_history.append(f"Action: {action}")
        self.current_history.append(f"Observation: {observation}")
```

这 30 行就是 ReAct 的心脏，五步节拍清晰可数。最值得注意的是**没写出来的部分**：`_parse_output`、`_parse_action`、`_parse_action_input` 三个解析方法在本文件中查无定义——它们全部继承自框架的 `ReActAgent` 基类。第四章我们为这些正则头疼了半章，如今它们成了基类的"标配肌肉"。

循环的记忆机制也和第四章一脉相承：LLM 本身无状态，每一步都把**全部** Action/Observation 历史重新塞进提示词，靠"复读"维持推理连续性。

第 96-99 行是保险丝熔断后的兜底：超过 `max_steps` 仍没等到 `Finish`，就体面认输（"抱歉，我无法在限定步数内完成"）并照常写入对话历史——**Agent 可以失败，但不能失联。**

---

## 🎯 剥完之后：它在本章的位置

`MyReActAgent` 是本章"扩展 Agent 层"的进阶示范，也是全书的一次首尾呼应：

- **对第四章**：同一个 ReAct 范式，裸写 → 框架化。对比两个文件你能精确看到框架替你消化了什么（解析、工具箱、历史管理），又把什么留给你定制（提示词、循环策略、步数）。
- **对本章**：与 `MySimpleAgent` 构成两种工具调用哲学的对照——SimpleAgent 把工具调用**藏进聊天**（轻协议），ReAct 把工具调用**立为纪律**（每步必须 Thought + Action，全程留痕可审计）。
- **对测试**：`test_react_agent.py` 用数学题、搜索题、复合推理题三连测它，还验证了 `custom_prompt` 注入——那是理解本文件的最佳"运行时注脚"。
