# Chapter 12 代码剥洋葱讲解

对 `code/chapter12/` 下九个示例脚本与 `data_generation/` 子目录的逐层拆解。每篇文档都采用"剥洋葱"结构：从最外层的**这个文件在干什么**开始，一层层剥到最核心的**那几行关键代码**，最后回到全局，说明它在智能体评估版图中的位置。

## 推荐阅读顺序

先看"为什么要评估"，再学两个现成基准（BFCL 考手、GAIA 考脑），最后学"没有标准答案时怎么评"：

| 顺序 | 文档 | 源文件 | 主题 | 一句话 |
|---|---|---|---|---|
| 1 | [01_basic_agent_example.md](01_basic_agent_example.md) | `01_basic_agent_example.py` | 为何需要评估 | 全章的被测对象原型：能跑 ≠ 跑得好 |
| 2 | [bfcl_evaluation.md](bfcl_evaluation.md) | `02/03/04_bfcl_*.py` | BFCL 函数调用基准 | AST 匹配判分：宽容表面差异，严格实质差异 |
| 3 | [gaia_evaluation.md](gaia_evaluation.md) | `05/06_gaia_*.py` | GAIA 通用助理基准 | 准精确匹配：官方提示词与归一化规则两头夹击 |
| 4 | [data_generation_metrics.md](data_generation_metrics.md) | `07/08/09_data_generation_*.py` | LLM Judge 与 Win Rate | 没有标准答案就用两把尺子：绝对分 + 相对胜率（理想 50%） |
| 5 | [data_generation_pipeline.md](data_generation_pipeline.md) | `data_generation/*.py` | 数据合成流水线实战 | 造题 → 机器质检 → 人工终检的完整闭环工程 |

## 一条主线

本章讲的是**智能体评估**：先用现成的基准考试，再学会自己出考卷。

> **BFCL**（考"手"：单步函数调用，AST 匹配判分，便宜而客观）
> **GAIA**（考"脑+手+腿"：多步真实任务，归一化后精确匹配，昂贵而全面）
> **数据合成评估**（自己出题时的质检：LLM Judge 打绝对分 + Win Rate 对标真题 + 人工验证兜底）

贯穿全章的一条评估铁律：**分数只有在协议闭合时才可信**——BFCL 的系统提示词要求纯 JSON 恰好对应 AST 解析器的输入，GAIA 的官方提示词逐条镜像判分器的归一化规则，LLM Judge 的评分维度与人工验证界面的四个滑块一字不差。协议的一端写在提示词里，另一端写在判分器里，两头对不上，测出来的就不是能力而是格式运气。

> 前置阅读：chapter4 的讲解（`code/chapter4/讲解/`）解释了智能体本身怎么构建；本章把智能体当作现成的"被测对象"，关注怎么给它出题、判分、下结论。
