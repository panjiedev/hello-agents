# data_generation 子目录剥洋葱式讲解

> 源文件：`code/chapter12/data_generation/` 下的 `aime_generator.py`（462 行）、`run_complete_evaluation.py`（315 行）、`step1_generate_only.py`（46 行）、`step2_evaluate_only.py`（288 行）、`human_verification_ui.py`（255 行）
> 一句话概括：**一座完整的"造题工厂"：AIMEGenerator 负责生产（few-shot 生成 + 断点续传 + 限速），两个评估工具负责质检（LLM Judge + Win Rate），Gradio 界面负责终检（人工验证）——生成、评审、把关的闭环在这里全部落地。**

---

## 🧅 第一层（最外层）：这个目录在干什么？

上一篇（`data_generation_metrics.md`）讲了两把尺子的原理，这个目录是把尺子装进流水线的工程实现。五个文件各司其职：

| 文件 | 角色 |
|---|---|
| `aime_generator.py` | 生产车间：批量生成 AIME 风格题目 |
| `run_complete_evaluation.py` | 总装线：生成 → LLM Judge → Win Rate → 综合报告，一条龙 |
| `step1_generate_only.py` / `step2_evaluate_only.py` | 总装线拆成两段：先生成后评估，中间可以人工介入 |
| `human_verification_ui.py` | 终检台：Gradio Web 界面人工打分 |

目录里的 `generated_data/` 和 `evaluation_results/` 是流水线的真实产物（生成的题目 JSON、评估报告 Markdown），可以直接翻看感受输出长什么样。

---

## 🧅 第二层：生成提示词——把"AIME 风格"写成规格书（aime_generator.py 第 23-46 行）

```python
GENERATION_PROMPT = """You are a professional mathematics competition problem designer...

AIME Problem Characteristics:
1. Answer: An integer between 0 and 999
2. Topics: Algebra, Geometry, Number Theory, Combinatorics, Probability, etc.
3. Style: Requires multi-step reasoning, but no advanced theory
4. Difficulty: Medium to hard (similar to AIME problems 6-9)
...
```json
{"problem": "...", "answer": 123, "solution": "...", "topic": "Algebra"}
```"""
```

提示词把"AIME 风格"翻译成四条可验证的规格：**答案是 0-999 的整数**（AIME 的著名特征，也让后续校验有章可循）、主题范围、多步推理但不用高等理论、难度对标第 6-9 题。输出照例锁死为 JSON。

更妙的是 `_build_prompt`（第 134-178 行）的 **few-shot 升级**：初始化时从 HuggingFace 加载 963 道 1983-2025 年真题（第 80-107 行），每次生成随机抽一道塞进提示词做风格参照——同时用三处强调 "completely different"、"do not copy the content" 防止模型照抄。**拿真题喂风格、靠措辞防抄袭**，这是数据合成最常用的一对平衡术。

---

## 🧅 第三层：解析响应——LaTeX 与 JSON 的战争（aime_generator.py 第 180-229 行）

数学题里全是 `\frac`、`\sqrt`，而 `\f`、`\s` 在 JSON 里是非法转义——模型输出的 JSON 十有八九直接 `json.loads` 会炸。`_parse_response` 的应对分三段：先剥掉 ```` ```json ```` 围栏（第 185-190 行）；直接解析失败后启动**正则修复**（见洋葱心）；解析成功后还要做业务校验（第 215-227 行）：

```python
answer = int(problem_data.get("answer", 0))
if not (0 <= answer <= 999):
    answer = max(0, min(999, answer))       # 钳制回合法区间
problem_data.setdefault("solution", "No solution provided")
```

答案越界不报废而是钳制，缺字段给默认值——生成流水线里**单条失败的代价是重跑一次昂贵的 API 调用**，所以能修则修、能兜则兜（`generate_single` 第 122-132 行还包了 3 次重试，彻底失败才返回占位题目）。

---

## 🧅 第四层：批量生成的三件工程套装（aime_generator.py 第 240-321 行）

`generate_batch` 是长时间跑批的教科书写法：

1. **断点续传**（第 264-273 行）：每生成一题就把全量列表写入 `checkpoint_*.json`；重启时读检查点、从第 N+1 题继续。30 道题 × 5 秒/题的任务，跑到第 29 题断网不用从头再来。
2. **主动限速**（第 280-288 行）：记录上次 API 调用时间，不足 `delay_seconds` 就补觉——不是被动等 429 报错重试，而是主动错峰。
3. **进度可视化**（第 277、305-310 行）：tqdm 进度条实时显示当前题目的主题、答案、耗时；日志用 `tqdm.write` 而不是 `print`，避免打乱进度条。

收尾时 `generate_and_save`（第 337-378 行）保存正式文件、生成统计报告（主题分布、答案分布）、删掉检查点——留下的 `generation_report_*.md` 让你不打开 JSON 也能一眼看出"几何题是不是生成得太多了"。

---

## 🧅 第五层：总装线与两段式（run_complete_evaluation.py、step1/step2）

`run_complete_evaluation.py` 把闭环串起来：生成（第 56-60 行）→ LLM Judge（第 80-93 行）→ Win Rate（第 104-116 行）→ 综合报告（第 129-141 行）。三个细节：

- 两个评估工具的调用形态是 `tool.run(params_dict)` **返回 JSON 字符串**（第 82-90 行），和 BFCL/GAIA 的 `tool.run(agent=...)` 不同——这里被评的是数据不是智能体；
- 每个评估都用独立的 `try/except` 包住（第 79-97 行）：Judge 挂了不影响 Win Rate，**部分结果好过没有结果**，报告生成处也用 `if llm_judge_result or win_rate_result` 兼容缺腿；
- 综合报告的结论判据要求 LLM Judge 分数与 Win Rate **双达标**（第 252-257 行，上一篇的洋葱心），并把下一步行动（人工验证命令）直接写进报告。

`step1_generate_only.py` / `step2_evaluate_only.py` 则是同一条线的**手动挡**：生成很贵、评估也不便宜，拆开后可以只重跑评估、或在两步之间先人工筛掉明显的废题再花评估钱。step2 的评估与报告代码和总装线几乎逐行相同——教学仓库为了每个脚本可独立运行而接受的重复，工程上应抽成共享模块。

---

## 🧅 第六层：人工验证——人和 LLM 用同一张评分表（human_verification_ui.py）

Gradio 界面的核心是 `verify_problem`（第 74-120 行）：

```python
self.verifications[problem_id] = {
    "scores": {"correctness": ..., "clarity": ...,
               "difficulty_match": ..., "completeness": ...},
    "total_score": (correctness + clarity + difficulty_match + completeness) / 4,
    "status": status,          # approved / rejected / needs_revision
    ...
}
self._save_verifications()
```

注意四个滑块（第 181-184 行）的维度**和 LLM Judge 的评估提示词一字不差**——正确性、清晰度、难度匹配、完整性。这不是巧合：人和机器用同一张量表打分，人工结果才能直接用来**校准** LLM Judge（如果人给 3 分机器给 5 分，说明阅卷模型偏松）。验证结果实时落盘到 `*_verifications.json`（每次提交即保存，第 118 行），`get_statistics`（第 134-161 行）随时汇总通过/拒绝/需修改的比例。

---

## 🧅 洋葱心：一行正则救活 LaTeX JSON（aime_generator.py 第 203 行）

```python
fixed_json_str = re.sub(r'(?<!\\)\\(?!["\\/bfnrtu])', r'\\\\', json_str)
```

整个目录最浓缩的一行。拆开这个正则：

- `(?<!\\)\\` —— 匹配一个**前面不是反斜杠**的反斜杠（避免把已转义的 `\\` 改成四个）；
- `(?!["\\/bfnrtu])` —— 且**后面不是 JSON 合法转义字符**（`\"`、`\\`、`\n`、`\t`、`\uXXXX` 等要放过）；
- 命中的就是 `\frac`、`\sqrt` 这类"LaTeX 合法、JSON 非法"的孤儿反斜杠，统一补成 `\\frac`。

它体现了数据合成流水线的一条生存法则：**LLM 的输出永远处在"几乎合法"的状态，解析层必须比协议更宽容一档**。先按标准解析，失败再定向修复，仍失败才打印原文报错（第 205-212 行）——三段式的防御让 963 次生成不至于折在第 7 次的一个 `\frac` 上。

---

## 🎯 剥完之后：它在本章的位置

这个目录是全章唯一"从零造轮子"的部分：BFCL、GAIA 用的是别人的题库和判分器，这里则完整走了一遍**造题（生成器）→ 机器质检（双指标）→ 人工终检（Gradio）→ 报告归档**的闭环。它同时是 07/08/09 三个示例脚本的引擎室，也是一个可以整体搬走的模板——把 `GENERATION_PROMPT` 换成法律问答、把 AIME 真题换成你的领域数据集，同一条流水线就能为任何垂直领域生产带质检报告的评估数据。
