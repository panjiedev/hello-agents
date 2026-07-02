# GAIA 评估两部曲剥洋葱式讲解

> 源文件：`code/chapter12/05_gaia_quick_start.py`（86 行）、`06_gaia_best_practices.py`（149 行）
> 一句话概括：**用 GAIA（General AI Assistants）基准考察智能体的"综合素质"——466 个真实世界问题、三级难度，判分靠"归一化后精确匹配"，而官方系统提示词就是归一化规则的镜像。**

---

## 🧅 第一层（最外层）：这两个文件在干什么？

BFCL 考的是单步的"函数调用写没写对"；GAIA 考的是**多步的真实任务**：查资料、算数字、读文件、跨来源推理，最后给出一个简短的确定答案。它由 Meta AI 和 Hugging Face 联合推出，分三级难度（Level 1 简单 → Level 3 困难），是受限数据集——要先在 HuggingFace 上申请权限、配好 `HF_TOKEN` 才能跑。

两个文件的分工：`05` 演示最小可用的评估调用；`06` 演示**怎么把有限的评估预算花在刀刃上**（GAIA 每样本要 1-5 次 API 调用，比 BFCL 贵得多）。

---

## 🧅 第二层：官方系统提示词——评分协议的另一半（05 第 19-23 行）

```python
GAIA_SYSTEM_PROMPT = """You are a general AI assistant. ... finish your answer with the following template: FINAL ANSWER: [YOUR FINAL ANSWER].
YOUR FINAL ANSWER should be a number OR as few words as possible OR a comma separated list ...
If you are asked for a number, don't use comma to write your number neither use units such as $ or percent sign ...
If you are asked for a string, don't use articles, neither abbreviations ..."""
```

注释里反复强调"**必须使用官方提示词**"，原因剥开来看很精妙：这段提示词的每一条格式要求，都和判分器的**归一化规则一一对应**（见洋葱心）。提示词让模型"数字不要写逗号和单位"，判分器归一化时恰好也"去掉逗号和单位"——两头夹击，最大限度消灭"答案对了但格式不对"的冤案。`FINAL ANSWER:` 模板则是答案提取的锚点：判分器只看这个标记后面的内容，前面的推理过程随便写。

换掉这段提示词，你测出来的就不是"智能体能力"，而是"格式运气"。

---

## 🧅 第三层：一键评估与双指标（05 第 40-52 行）

```python
results = gaia_tool.run(
    agent=agent,
    level=1,              # 评估级别（1=简单，2=中等，3=困难）
    max_samples=2,
    export_results=True,  # 导出GAIA官方提交格式
    generate_report=True
)
print(f"精确匹配率: {results['exact_match_rate']:.2%}")
print(f"部分匹配率: {results['partial_match_rate']:.2%}")
```

和 BFCL 一样的 `tool.run()` 形态，但返回**两个匹配率**：`exact_match_rate` 是官方口径（归一化后完全相等才算对），`partial_match_rate` 是宽松口径（部分吻合也计入）。两者的差值是很好的诊断信号——差值大说明智能体"知道答案但说不利索"，该修的是输出格式而不是推理能力。`export_results=True` 会顺手产出 GAIA 官方排行榜的提交文件（`template_output/evaluation_results/gaia_official/` 里有现成样例）。

---

## 🧅 第四层：三个最佳实践——把评估当消费决策（06）

**实践 1：分级递进评估（第 43-60 行）**

```python
results_l1 = gaia_tool.run(agent, level=1, max_samples=10)
if results_l1['exact_match_rate'] > 0.6:
    results_l2 = gaia_tool.run(agent, level=2, max_samples=10)
    if results_l2['exact_match_rate'] > 0.4:
        results_l3 = gaia_tool.run(agent, level=3, max_samples=10)
```

逻辑像游戏闯关：Level 1 及格线 60%，过了才解锁 Level 2（及格线 40%）。背后是成本直觉——Level 1 都答不好的智能体，去跑 Level 3 只是烧钱验证"果然不行"。

**实践 2：小样本快测（第 70-73 行）**：每级只跑 2 个样本，用几毛钱先确认管道通、Token 有效、格式对，再上量。

**实践 3：结果解读（第 82-121 行）**：`interpret_results` 把裸数字翻译成结论和行动建议，注意**各级的"优秀线"是递减的**——Level 1 要 60%，Level 2 只要 40%，Level 3 只要 20%（GAIA 论文里人类能到 92%，而当年 GPT-4 加插件也只有 15%，Level 3 的 20% 已是强者）。给的建议也分层：Level 1 差先查提示词和答案提取（工程问题），Level 2/3 差再谈推理和工具链（能力问题）。

最后的**难度递进分析**（第 138-148 行）是一个便宜的健全性检查：正常应有 L1 > L2 > L3，如果倒挂，先怀疑评估本身（样本太少、数据偏差），再怀疑智能体。

---

## 🧅 洋葱心：准精确匹配——先归一化，再硬比对

GAIA 的判分算法（实现在框架的 `evaluation/gaia/quasi_exact_match.py`）一行公式：

$$\text{Quasi\_Exact\_Match}(A_{\text{pred}}, A_{\text{true}}) = 1 \iff \mathcal{N}(A_{\text{pred}}) = \mathcal{N}(A_{\text{true}})$$

灵魂全在归一化函数 $\mathcal{N}$，按答案类型分三路：

| 类型 | 规则 | 例子 |
|---|---|---|
| 数字 | 去逗号分隔符、去单位符号 | `"$1,234.56"` → `"1234.56"` |
| 字符串 | 转小写、去冠词、压空格、去末尾标点 | `"The United States"` → `"united states"` |
| 列表 | 按逗号拆分、逐项归一化、**排序**后重连 | `"Paris, London, Berlin"` → `"berlin,london,paris"` |

归一化之后就是最朴素的字符串相等。对比 BFCL 的 AST 匹配，思路殊途同归：**把所有"不该扣分的表面差异"在比较前折叠掉，剩下的差异才是真差异**。而第二层讲的官方系统提示词，正是把这套折叠规则**提前告诉模型**——协议的两端，一端写在提示词里，一端写在判分器里。

---

## 🎯 剥完之后：它在本章的位置

GAIA 补上了 BFCL 缺的那半边：BFCL 考"手"（单步工具调用），GAIA 考"脑+手+腿"（多步推理、工具组合、真实世界信息获取）。两者合起来，回答了"现成基准怎么用"。但现成基准总有一天不够用——题目会泄漏进训练集，垂直领域没有现成题库。到那时就得自己造题，并回答一个更难的问题：**造出来的题质量怎么评？**这就是下一篇数据合成评估（`data_generation_metrics.md`）的主题。
