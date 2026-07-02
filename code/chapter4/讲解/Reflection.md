# Reflection.py 剥洋葱式讲解

> 源文件：`code/chapter4/Reflection.py`（163 行）
> 一句话概括：**实现 Reflection（反思）范式——让 LLM 扮演"程序员"写代码、再扮演"评审专家"挑毛病、再根据反馈重写，在自我批评的循环中迭代出更好的答案。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

前两个范式（ReAct、Plan-and-Solve）都是"一遍过"：答案生成即终点。Reflection 引入了一个人人熟悉的机制——**打草稿、自我检查、修改重写**：

```
任务：写一个函数，找出 1 到 n 之间所有素数

初稿   → 试除法（O(n√n)）
反思   → "评审员"：试除法效率低，建议用埃拉托斯特尼筛法
优化   → 第二稿：筛法（O(n log log n)）
反思   → "评审员"：已是最优算法，无需改进
结束   ✅
```

运行本文件（`__main__`，第 149-162 行）就是对这个素数任务跑最多 2 轮"反思-优化"循环。整个过程**没有外部工具、没有真的运行代码**——评审也是 LLM 做的，靠的是它对算法知识的"内省"。

---

## 🧅 第二层：`Memory`——为反思准备的记忆（第 7-45 行）

```python
self.records: List[Dict[str, Any]] = []
# 每条记录：{"type": "execution" 或 "reflection", "content": "..."}
```

一个列表存两种记录：**execution**（某一版代码）和 **reflection**（某一轮评审意见），天然按时间交替排列。对外提供两个读取视角，恰好对应两种消费者：

- **`get_trajectory()`** → 把全部记录格式化成"上一轮尝试 (代码) / 评审员反馈"交替的长文本。完整轨迹，给需要全程上下文的场合。
- **`get_last_execution()`** → 用 `reversed()` 从后往前找第一条 execution，即**最新版代码**。给"评审"和"优化"环节用——它们只关心当前版本。

和 ReAct 用一个字符串列表存历史相比，这里的记忆**带类型、可查询**——这是走向第八章"记忆系统"的第一步：记忆不只是流水账，而是可以按需检索的结构。

---

## 🧅 第三层：三个提示词 = 三个人格（第 50-95 行）

Reflection 的全部戏剧性都在这三段提示词里，同一个 LLM 轮流戴三顶帽子：

| 提示词 | 人格 | 输入 | 要求输出 |
|---|---|---|---|
| `INITIAL_PROMPT` | 资深 Python 程序员 | 任务 | 直接输出代码 |
| `REFLECT_PROMPT` | **极其严格的**代码评审专家 | 任务 + 当前代码 | 指出算法瓶颈，或"无需改进" |
| `REFINE_PROMPT` | 程序员（收到反馈后） | 任务 + 上版代码 + 反馈 | 优化后的新代码 |

反思提示词是三者中最讲究的，剥开看三处措辞：

1. **"极其严格"、"对性能有极致的要求"**——不是修辞。LLM 有强烈的"顺从倾向"，温和的评审员会说"整体不错"；只有把人格推向苛刻，才能逼出真问题。
2. **"专注于找出算法效率上的主要瓶颈"**——把评审面收窄到单一维度。什么都评 = 什么都评不深，而且反馈发散会让下一轮优化无所适从。
3. **"如果代码在算法层面已经达到最优，才能回答'无需改进'"**——这句话是整个循环的**出口协议**：终止信号被明确定义成一个可被程序检测的暗号，同时"才能"二字抬高了说出它的门槛，防止评审员偷懒提前收工。

甚至提示词里直接埋了提示：**"例如，使用筛法替代试除法"**——作者预判了初稿大概率是试除法，把期望的改进方向写进了评审员的知识里。教学代码的贴心，也是提示词工程"引导而非放任"的示范。

---

## 🧅 洋葱心：`run` 的迭代循环与停机条件（第 103-140 行）

```python
# 1. 初稿
initial_code = self._get_llm_response(INITIAL_PROMPT_TEMPLATE.format(task=task))
self.memory.add_record("execution", initial_code)

# 2. 反思-优化循环
for i in range(self.max_iterations):
    last_code = self.memory.get_last_execution()
    feedback = self._get_llm_response(REFLECT_PROMPT.format(task=task, code=last_code))
    self.memory.add_record("reflection", feedback)

    if "无需改进" in feedback or "no need for improvement" in feedback.lower():
        break                                    # ← 停机条件

    refined_code = self._get_llm_response(REFINE_PROMPT.format(...))
    self.memory.add_record("execution", refined_code)

final_code = self.memory.get_last_execution()   # 无论何时停，都取最新版
```

节奏是：**执行一次，然后（反思 → 判停 → 优化）转圈**。两处最值得剥：

### 停机条件：`"无需改进" in feedback`

用**字符串包含**来检测终止信号——简陋但有效，前提正是反思提示词里那句出口协议：先教模型"满意就说这四个字"，再用 `in` 来抓。它和 ReAct 的 `Finish[...]`、Plan-and-Solve 的 ` ```python ` 前后缀是同一件事：**提示词定协议，代码做解析，两头必须严丝合缝**。中英双语检查（`no need for improvement`）是对模型偶尔飙英文的兜底。

双保险停机和 ReAct 如出一辙：**质量收敛**（评审说满意）或 **`max_iterations` 用尽**（防止评审员永远不满意，无限烧 token）。

### 精妙的收尾：`get_last_execution()`

无论循环从哪个出口退出——评审满意、轮次用尽、甚至最后一轮只反思了还没来得及优化——**记忆里最新的那版代码就是最终交付物**。把"取结果"统一收敛到记忆查询上，省去了在每个出口各写一份返回逻辑。

---

## 🎯 剥完之后：三大范式同框

至此 chapter4 的三种智能体范式集齐，它们回答的是同一个问题的三个阶段——**如何组织 LLM 的思考**：

| 范式 | 一句话 | 类比 | 代价 |
|---|---|---|---|
| ReAct | 边想边做，用观察修正方向 | 侦探破案 | 缺全局观 |
| Plan-and-Solve | 先谋后动，按图施工 | 工程师画图纸 | 计划僵化 |
| Reflection | 做完再审，迭代精进 | 作家改稿 | 多倍 token 开销 |

三者并不互斥：真实的生产级智能体往往是组合体——**用 Plan 定框架，用 ReAct 执行每个步骤（可调工具），用 Reflection 审查最终产出**。而本文件的两个软肋也指向了后续章节：评审靠 LLM"感觉"而非真实运行（→ 让智能体执行代码、用测试结果反思，即 Reflexion 论文的做法）；记忆只活一次任务（→ 第八章的长期记忆系统）。
