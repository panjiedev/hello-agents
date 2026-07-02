# 数据合成评估三部曲剥洋葱式讲解

> 源文件：`code/chapter12/08_data_generation_llm_judge.py`（168 行）、`09_data_generation_win_rate.py`（171 行）、`07_data_generation_complete_flow.py`（119 行）
> 一句话概括：**当被评的东西没有标准答案（LLM 生成的 AIME 数学题），就换两把新尺子——LLM Judge 打绝对分（4 维度 1-5 分），Win Rate 打相对分（和真题捉对厮杀，理想胜率是 50%）。**

---

## 🧅 第一层（最外层）：这三个文件在干什么？

BFCL 和 GAIA 都有标准答案，判分是确定性算法。但本节的场景变了：用 LLM 生成 AIME 风格的数学竞赛题，**"这道生成题好不好"没有 ground truth 可比**。怎么办？本节给出智能体领域的主流答案：

- `08`：**LLM Judge**——请一个强模型（gpt-4o）当阅卷老师，按固定量表打绝对分；
- `09`：**Win Rate**——让生成题和 AIME 真题匿名 PK，请 LLM 判哪个更好，统计胜率；
- `07`：把生成 + 两种评估 + 人工验证串成**完整闭环**的入口脚本。

阅读顺序建议 08 → 09 → 07：先懂两把尺子，再看流水线。

---

## 🧅 第二层：LLM Judge——把主观评价拆成量表（08）

核心调用只有三行（第 42-57 行）：

```python
llm = HelloAgentsLLM(model_name="gpt-4o")
judge = LLMJudge(llm=llm)
result = judge.evaluate_single(problem)   # problem 含 problem/answer/solution
```

`LLMJudge` 内部的评估提示词（在框架里，全文见教程 12.4.3 节）是这套方法的灵魂：

```
请从以下4个维度评分（1-5分）：
1. 正确性 (Correctness)：数学逻辑是否正确，答案是否准确
2. 清晰度 (Clarity)：问题表述是否清晰，解答是否易懂
3. 难度匹配 (Difficulty Match)：难度是否符合AIME标准（中等偏难）
4. 完整性 (Completeness)：解答步骤是否完整，是否包含必要的推理

请按以下JSON格式输出：
{"correctness": 5, "clarity": 4, "difficulty_match": 4, "completeness": 5, "comments": "..."}
```

剥开看有三个设计决策：**拆维度**（不问笼统的"好不好"，问四个可独立判断的具体问题，降低评审模型的发挥空间）；**限定量表**（1-5 整数分，可聚合可比较）；**强制 JSON**（又是"提示词定协议，代码做解析"）。`comments` 字段则保留了定性反馈——分数用来筛选，评语用来改进。

脚本随后做朴素的聚合（第 76-87 行：逐维度求平均）和阈值分级（第 91-98 行）：均分 ≥4.0 优秀可直接用，≥3.0 良好但建议人工审核，再往下重新生成。框架层面还定义了两个补充指标：**及格率**（均分 ≥3.5 的题目占比）和**优秀率**（≥4.5 占比）——平均分看整体水平，及格率保底线，优秀率看上限。

---

## 🧅 第三层：Win Rate——相对比较，50% 才是满分（09）

LLM Judge 的绝对分有个软肋：评审模型可能整体偏松或偏严（4.2 分到底算好算坏？）。Win Rate 用**相对比较**绕开标定问题（第 49-68 行）：

```python
dataset = AIDataset()
reference_problems = dataset.load()            # 963 道 AIME 真题
evaluator = WinRateEvaluator(llm=llm, reference_problems=reference_problems)
results = evaluator.evaluate(generated_problems=generated_problems,
                             num_comparisons=20)
```

每次对比，评审模型看到一道生成题和一道随机抽取的真题，按对比提示词（框架内）判 `"winner": "A" 或 "B" 或 "Tie"` 并给理由。20 次对比后统计：

$$\text{Win Rate} = \frac{\text{Wins}}{\text{Total}},\quad \text{Win} + \text{Loss} + \text{Tie} = 100\%$$

最反直觉也最重要的一点在判读阈值（第 89-96 行）：

```python
if 0.45 <= win_rate <= 0.55:
    print("✅ 优秀 - 生成质量接近AIME真题水平")
```

**45%-55% 才是优秀，而不是越高越好。**对手是官方真题，打成平手（≈50%）就意味着评审模型分不出生成题和真题的质量差异——这正是数据合成的终极目标。Win Rate 显著高于 50% 反而是警报：大概率不是你的题超越了 AIME 命题组，而是评估存在偏差（比如评审模型偏爱某种表述风格）。

第 103-109 行打印的 `comparisons` 明细（生成题、真题、判决、理由）是宝贵的改进素材：连输几局的理由如果都是"难度不足"，下一轮生成提示词就知道该往哪改。

---

## 🧅 第四层：完整流程入口——薄封装与一个坑（07）

`07` 本体极薄（第 26-50 行）：解析命令行参数（生成数量、每题延迟秒数），然后转调子目录里的真正实现：

```python
from data_generation.run_complete_evaluation import main
...
main(num_problems, delay_seconds)
```

它的价值是把闭环的四步昭示出来：**生成 AIME 题 → LLM Judge 绝对分 → Win Rate 相对分 → 人工验证**。真正的流水线代码在 `data_generation/` 子目录（见 `data_generation_pipeline.md`）。

一个需要留意的坑：`run_complete_evaluation.py` 里的 `main()` 定义为**无参函数**（它自己解析 `sys.argv`），而这里以 `main(num_problems, delay_seconds)` 带参调用，直接运行会抛 `TypeError`。绕开方式是改为导入并调用带参的 `run_complete_evaluation(num_problems, delay_seconds)`，或者干脆直接运行子目录脚本：`python data_generation/run_complete_evaluation.py 30 3.0`。

---

## 🧅 洋葱心：两把尺子为什么要同时用？

单看任何一把尺子都可能被骗，合起来才构成交叉验证。子目录 `run_complete_evaluation.py` 第 252-257 行的综合结论判据，就是这个思想的落点：

```python
if overall_avg_score >= 4.5 and overall_win_rate >= 0.48:
    report += "✅ **结论**: 生成数据质量**优秀**，达到或超过AIME真题水平。\n"
elif overall_avg_score >= 4.0 and overall_win_rate >= 0.45:
    report += "✅ **结论**: 生成数据质量**良好**，接近AIME真题水平。\n"
```

**LLM Judge 分数（绝对轴）和 Win Rate（相对轴）必须同时达标**：只有 Judge 高分可能是阅卷偏松，只有 Win Rate 接近 50% 可能是"和真题一样平庸地难以区分"。两轴交叉，再加最后一道人工验证兜底，才敢下"接近真题水平"的结论。

---

## 🎯 剥完之后：它在本章的位置

这三个文件把本章从"用别人的考卷"推进到"自己出考卷、并给出卷质量打分"。方法论上它们也回答了一个更普遍的问题——**任何没有标准答案的智能体输出（写作、代码评审、对话质量）都可以套用这套组合拳**：LLM Judge 定绝对量表、Win Rate 对标参照系、人工验证抽查兜底。三者的工程化实现（生成器、断点续传、Gradio 验证界面），见下一篇 `data_generation_pipeline.md`。
