# 05_grpo_training.py 剥洋葱式讲解

> 源文件：`code/chapter11/05_grpo_training.py`（334 行）
> 一句话概括：**GRPO 强化学习训练——让模型对同一道题生成一组解答，用奖励函数打分后组内比较，"比同伴好的被强化，比同伴差的被抑制"，无需价值网络的轻量级 RLHF。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

SFT 教会了模型"怎么写"，但没教会"怎么写对"——临摹字帖不等于会做新题。GRPO（Group Relative Policy Optimization，组相对策略优化）接棒：不再给模型看标准解答，而是让它**自己做题、自己试错**，奖励函数（02 篇）判对错，答对的行为被强化。

表面上，这个文件和 `04_sft_training.py` 长得像双胞胎——同样 6 个示例、同样的配置字典结构，甚至示例 1 的最小配置只差一个字：

```python
config = {
    "action": "train",
    "algorithm": "grpo",     # ← 唯一的区别：sft 改成 grpo
    "model_name": "Qwen/Qwen3-0.6B",
    ...
}
```

但配置的相似掩盖着训练循环的天壤之别——这正是要往下剥的。

---

## 🧅 第二层：GRPO 的训练循环 vs SFT 的训练循环

把两种训练的一步（step）摆在一起：

```
SFT 的一步:
  取一批 (prompt, 人写的completion) → 前向传播 → 逐token交叉熵 → 更新

GRPO 的一步:
  取一批 prompt
  → 模型对每个 prompt 采样生成 G 个解答     ← 多了"生成"（rollout）
  → 奖励函数给 G 个解答逐个打分             ← 多了"评分"
  → 组内归一化算优势: Âᵢ = (rᵢ - mean(r)) / std(r)   ← 多了"比较"
  → 按优势加权的策略梯度更新
```

三个本质差异：

1. **监督信号的来源**：SFT 的信号来自数据集里人写的答案（密集、逐 token）；GRPO 的信号来自奖励函数对模型自产文本的打分（稀疏、整句一个分数）。
2. **训练中要不要生成**：SFT 只做前向传播；GRPO 每一步都要先让模型完整地生成 G 份解答——这是它慢、且吃显存的根源。
3. **"组相对"是 GRPO 的招牌**：PPO 需要额外训练一个价值网络（critic）来估计基线，GRPO 直接用**同组 G 个解答的平均奖励当基线**——同一道题生成 8 个答案，谁比组平均分高谁被强化。省掉整个 critic 模型，显存和工程复杂度双降，这是 DeepSeek 系列把它带火的原因。

---

## 🧅 第三层：配置数字里的算法指纹（第 64-84 行）

示例 2 的标准配置，每个数字都在泄露 GRPO 的脾气：

```python
config = {
    "algorithm": "grpo",
    "model_name": "Qwen/Qwen3-0.6B",  # 或 "./output/sft_standard"
    "max_samples": 500,       # GRPO通常使用较少样本
    "num_epochs": 3,
    "batch_size": 2,          # GRPO需要更多显存
    "learning_rate": 1e-5,    # 比SFT小10倍
    ...
}
```

对照 SFT 篇的标准配置（1000 样本、batch 4、lr 5e-5）逐项读：

- **`batch_size: 2`（SFT 的一半）**——每个 prompt 要生成 G 个解答，实际前向的序列数是 `batch × G`，显存开销成倍放大；
- **`learning_rate: 1e-5`（SFT 的 1/10）**——RL 的梯度来自采样估计，噪声远大于监督学习；而且策略一旦被大步长带偏，生成质量崩了，后续采样全是垃圾，训练会雪崩式失稳。小步慢走是 RL 的保命纪律；
- **`max_samples: 500`（SFT 的一半）**——GRPO 每个样本要生成并评分 G 次，单样本的"榨取量"更高，靠质量不靠数量。

---

## 🧅 洋葱心：两级火箭的点火顺序（第 144-197 行）

示例 4 是全章的剧本核心——SFT 和 GRPO 的接力：

```python
# 步骤1: SFT训练
sft_config = {
    "algorithm": "sft",
    "model_name": "Qwen/Qwen3-0.6B",          # 从基座起步
    "output_dir": "./output/pipeline_sft",
}
# 步骤2: GRPO训练
grpo_config = {
    "algorithm": "grpo",
    "model_name": "./output/pipeline_sft",    # ← 洋葱心：用SFT模型接棒
    "learning_rate": 1e-5,
}
```

第 179 行 `"model_name": "./output/pipeline_sft"`——`model_name` 从 HuggingFace 模型名换成了**上一阶段的输出目录**。就这一行，把两个独立的训练拼成了流水线。

为什么必须这个顺序？回到 GRPO 的机制：它靠"组内比较"产生信号，前提是**组里得有好有坏**。如果直接从基座起步，模型输出格式混乱，奖励函数（依赖 `Final Answer:` 提取答案）几乎全判 0 分——组内全是零分，mean=0、std=0，优势无从算起，训练原地踏步。SFT 先把格式钉死、把正确率抬到"有对有错"的区间，GRPO 的组内比较才有梯度可爬。**SFT 负责冷启动，GRPO 负责登顶**。

---

## 🎯 剥完之后：它在本章的位置

GRPO 是本章标题"Agentic RL"里那个 RL 的落点，也是把前面所有零件拧在一起的总装车间：

| 依赖 | 提供什么 |
|---|---|
| `01_dataset_loading.py` 的 RL 格式 | 带聊天模板的 prompt + `ground_truth` |
| `02_reward_functions.py` | 训练循环里的打分裁判（默认 accuracy，可换，见示例 5） |
| `03_lora_configuration.py` | 让"生成 + 训练"同时驻留显存成为可能 |
| `04_sft_training.py` 的产物 | 冷启动的起点模型 |

一句话记住 SFT 与 GRPO 的分工：**SFT 是"照着答案学"，GRPO 是"做题对答案、组内排名定奖惩"**。它们的合体将在 `06_complete_pipeline.py` 里被编排成一条自动化流水线。
