# 03/04_note_tool 剥洋葱式讲解

> 源文件：`code/chapter9/03_note_tool_operations.py`（134 行）+ `code/chapter9/04_note_tool_integration.py`（277 行）
> 一句话概括：**NoteTool 是智能体的"外部记事本"——先学会它的读写协议（03），再看笔记如何被检索、转成上下文包、注回模型视野，形成完整的外部记忆回路（04）。**

---

## 🧅 第一层（最外层）：这两个文件在干什么？

上下文窗口是易失的：会话一结束，模型"脑子里"的一切归零。MemoryTool 的向量记忆能缓解，但它是模糊检索，不适合存"精确状态"——比如"第一阶段完成了 85%，卡在依赖冲突上"。

NoteTool 的答案朴素而有效：**把状态写成 Markdown 文件**。每条笔记 = YAML 头（id / title / type / tags / 时间戳）+ Markdown 正文，外加一个 `notes_index.json` 索引。人能读、git 能管、程序能查、模型能引用。

- `03_note_tool_operations.py` 走一遍笔记的完整生命周期：create → read → update → search → list → summary →（delete）；
- `04_note_tool_integration.py` 把 NoteTool 接进 ContextBuilder，实现"上一轮写的笔记，下一轮自动出现在模型眼前"。

---

## 🧅 第二层：统一的调用协议——一个 dict 走天下（03 第 36-47 行）

```python
create_output_1 = notes.run({
    "action": "create",
    "title": "重构项目 - 第一阶段",
    "content": """## 完成情况
已完成数据模型层的重构,测试覆盖率达到85%。

## 下一步
重构业务逻辑层""",
    "note_type": "task_state",
    "tags": ["refactoring", "phase1"]
})
```

所有操作共用一个入口 `run(dict)`，用 `action` 字段分发——这正是工具能被 LLM 函数调用（Function Calling）驱动的前提：**接口越统一，模型越容易学会用它**。

`note_type` 是笔记的"语义类型"，本章约定了一套小型分类法：

| 类型 | 含义 | 典型内容 |
|---|---|---|
| `task_state` | 任务状态 | 完成了什么、进度百分比 |
| `blocker` | 阻塞问题 | 卡在哪、影响范围 |
| `action` | 行动计划 | 下一步做什么 |
| `conclusion` | 结论 | 分析得出的判断 |

分类不是摆设——第 104-110 行 `list` 时按 `note_type="blocker"` 过滤，04 中检索时 blocker 优先注入上下文。**类型即优先级**。

---

## 🧅 第三层：一个不起眼但关键的函数——extract_note_id（03 第 18-23 行）

```python
def extract_note_id(output: str) -> str:
    """从 NoteTool 的输出文本中提取 note_id"""
    match = re.search(r"ID:\s*(note_[0-9_]+)", output)
    if not match:
        raise ValueError(f"无法从输出解析 note_id:\n{output}")
    return match.group(1)
```

NoteTool 的 `run()` 返回的是**给人（和 LLM）看的文本**，不是结构化对象。程序想拿到刚创建笔记的 ID 去 read / update，就得用正则从文本里抠。这是 chapter4 那条铁律（"提示词定协议，代码做解析"）的工具版变体：**工具输出定协议，调用方做解析**——协议是 `ID: note_xxx` 这个固定格式。

后续操作全靠这个 ID 串起来：第 72-76 行 read、第 80-92 行 update（update 是全量覆盖正文，新内容里补记了"遇到依赖冲突"——笔记的演进就是状态的演进）。

---

## 🧅 第四层：外部记忆回路——检索、转包、注入（04 第 45-76 行）

04 的 `ProjectAssistant.run()` 把 NoteTool 和 ContextBuilder 焊在一起，六步一循环：

```python
# 1. 从 NoteTool 检索相关笔记
relevant_notes = self._retrieve_relevant_notes(user_input)
# 2. 将笔记转换为 ContextPacket
note_packets = self._notes_to_packets(relevant_notes)
# 3. 构建优化的上下文
optimized_context = self.context_builder.build(
    ..., additional_packets=note_packets
)
# 4. 调用 LLM
# 5. 如果需要,将交互记录为笔记
# 6. 更新对话历史
```

注意这个环的形状：**第 5 步写下的笔记，会在未来某轮的第 1 步被检索回来**。这就是"笔记作为外部记忆"的读写闭环——写入靠 `_save_as_note`，读出靠 `_retrieve_relevant_notes`，桥梁是 `additional_packets` 这个 ContextBuilder 的注入口。

检索策略（第 78-115 行）值得一看：不是单纯按相关性搜，而是**blocker 类型无条件优先捞 2 条 + 关键词搜索捞 3 条，合并去重**——"卡住的问题"比"相关的信息"更该被模型记住。

写入策略（第 188-208 行）则用最朴素的关键词路由决定笔记类型：

```python
if "问题" in user_input or "阻塞" in user_input:
    note_type = "blocker"
elif "计划" in user_input or "下一步" in user_input:
    note_type = "action"
else:
    note_type = "conclusion"
```

简陋但够用——分类的价值在于检索时的优先级，不在于分类本身多精确。

---

## 🧅 第五层：防御性编程的密集示范（04 第 117-171 行）

`_ensure_list_of_dicts` 和 `_notes_to_packets` 里堆满了兼容代码：返回值可能是 str / dict / list，逐一规范化；note_id 可能叫 `note_id` / `id` / `uuid`，逐一尝试；时间戳可能是数字 / ISO 字符串 / 缺失，逐一解析、最后 `datetime.now()` 兜底。

这不是啰嗦——**工具的输出格式是"软契约"**（可能随版本变化），而上下文构建绝不能因为一条笔记格式不对就整体崩掉。检索失败也只是 `print WARNING` 后返回空列表（第 113-115 行）：宁可这一轮少一条记忆，不可让对话中断。

---

## 🧅 洋葱心：笔记变成上下文包的七行（04 第 174-184 行）

```python
packets.append(ContextPacket(
    content=content,                    # "[笔记:标题]\n正文"
    timestamp=parsed_ts,
    token_count=len(content) // 4,      # 简单估算
    relevance_score=0.75,               # 笔记具有较高相关性
    metadata={"type": "note", "note_type": note_type, "note_id": note_id}
))
```

这七行是两个世界的转接头：**文件系统里的 Markdown（持久、无限容量）→ 上下文里的信息包（易失、寸土寸金）**。三个细节：

1. `relevance_score=0.75` 是**人为标定的先验**——笔记是自己昨天刻意写下的，天然比随机的历史消息更值得信任，直接给高分免去评分环节；
2. `token_count=len//4` 粗估即可——预算控制需要的是量级，不是精度；
3. `metadata` 保留 `note_id`——万一模型想深挖，还能顺着 ID 用 read 取全文。

一旦变成 ContextPacket，笔记就和对话历史、记忆平起平坐，一起参加 ContextBuilder 的评分与装配。**外部记忆的本质，就是给信息找一个不怕会话结束的家，再修一条随时回到上下文的路。**

---

## 🎯 剥完之后：它在本章的位置

NoteTool 补上了 ContextBuilder 管不到的时间维度：

| 组件 | 记忆时长 | 特点 |
|---|---|---|
| 对话历史 | 单会话 | 自动积累，会被评分淘汰 |
| MemoryTool | 跨会话 | 向量检索，模糊召回 |
| **NoteTool** | **跨会话/跨周** | **精确状态，人机共读，git 友好** |

`codebase_maintainer.py` 里它是三大工具之一（且按笔记类型给出 0.9/0.8/0.75/0.7 的差异化相关性）；`06_three_day_workflow.py` 的"跨会话连贯性"演示，底牌就是同一个笔记目录被两次会话共享。**记笔记，是长程智能体对抗遗忘的第一生产力。**
