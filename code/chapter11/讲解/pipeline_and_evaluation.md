# 06_complete_pipeline.py & 07_model_evaluation.py 剥洋葱式讲解

> 源文件：`code/chapter11/06_complete_pipeline.py`（262 行，配 `config.json`）+ `code/chapter11/07_model_evaluation.py`（335 行）
> 一句话概括：**把"数据 → SFT → 评估 → GRPO → 评估 → 存档"六个阶段编排成一条端到端流水线，并用"基线 < SFT < GRPO"的三方对比来验收训练是否真的有效。**

---

## 🧅 第一层（最外层）：这两个文件在干什么？

前面 01～05 每个文件只演示一个环节，都是"手动挡"。`06_complete_pipeline.py` 换成"自动挡"：一个 `AgenticRLPipeline` 类，`pipeline.run()` 一声令下，六个阶段依次执行，中途任何一步的产物自动流向下一步。

而训练完的模型好不好，不能靠感觉——`07_model_evaluation.py` 负责"验收"：在 GSM8K 测试集上跑准确率，并且必须**三方对比**（基线原始模型 / SFT 模型 / GRPO 模型），否则你无法回答"训练到底带来了多少提升"。

---

## 🧅 第二层：流水线的骨架——类 + 配置 + 阶段方法（06 第 21-43、195-222 行）

```python
class AgenticRLPipeline:
    def __init__(self, config_path="config.json"):
        self.rl_tool = RLTrainingTool()
        self.config = self.load_config(config_path)
        self.results = {}
```

三个成员就是流水线的全部状态：**工具**（执行者）、**配置**（从 `config.json` 读，改实验不改代码）、**结果簿**（`self.results`，每个阶段往里记一笔）。

`run()` 方法（第 195-222 行）是总调度，读它就能看清数据流：

```python
self.stage1_prepare_data()
sft_model_path = self.stage2_sft_training()        # 返回SFT模型路径
self.stage3_sft_evaluation(sft_model_path)         # 评它
grpo_model_path = self.stage4_grpo_training(sft_model_path)  # 在它之上继续训练
self.stage5_grpo_evaluation(grpo_model_path)
self.stage6_save_results()
```

**阶段之间靠返回值传递模型路径**——`stage2` 的返回值同时喂给 `stage3`（评估）和 `stage4`（GRPO 的起点），这正是 05 篇"两级火箭"的自动化版本。整个 `run()` 包在一个 `try/except` 里（第 197-222 行）：训练动辄几小时，失败必须显式报出来而不是静默吞掉。

值得注意的还有 `config.json` 里一个躺着没人用的字段：`"sft_accuracy_threshold": 0.40`——它暗示了流水线的进阶形态：**SFT 评估不达标就不该浪费算力去跑 GRPO**（质量门禁）。当前代码没实现这个拦截，是留给读者的扩展点。

---

## 🧅 第三层：评估为什么必须"三方对比"？（07 第 177-227 行）

07 的前四个示例分别评 SFT、GRPO、基线，示例 5 把它们合成一个循环：

```python
models = {
    "基线模型": "Qwen/Qwen3-0.6B",
    "SFT模型": "./output/quick_test/sft",
    "GRPO模型": "./output/quick_test/grpo"
}
for name, model_path in models.items():
    config = {"action": "evaluate", "model_path": model_path, "max_samples": 100}
```

评估的机制其实是奖励函数的复用：让模型在**测试集**（训练时没见过的题）上生成解答，用 02 篇的答案提取 + 数值比较判对错，返回 `accuracy` 和 `average_reward` 两个指标。

三方对比的每一条边都有独立含义：

- **基线 vs SFT**：模仿学习学到了多少（格式 + 套路的收益）；
- **SFT vs GRPO**：强化学习又榨出了多少（推理正确率的收益）；
- **基线 vs GRPO**：整条流水线的总账。

预期结果写在第 222 行：`基线模型 < SFT模型 < GRPO模型`。如果 SFT ≈ 基线，说明数据或格式有问题；如果 GRPO ≈ SFT，多半是奖励信号出了毛病（比如答案提取大面积失败）——**对比不只是报喜，更是排障的定位器**。

另外注意 07 示例 6 的一个工程好习惯（第 251-259 行）：评估前先 `os.path.exists()` 检查模型目录在不在，不在就提示"请先运行 00_quick_test.py"——依赖前置产物的脚本，应当把前置条件检查做在最前面。

---

## 🧅 洋葱心：结果落盘的三行（06 第 188-193 行）

```python
results_path = "training_results.json"
with open(results_path, 'w') as f:
    json.dump(self.results, f, indent=2)
```

看似平平无奇的三行，是整条流水线的"存在证明"。回想 00 篇的总协议——`RLTrainingTool` 的所有输出都是 JSON 字符串，每个阶段 `json.loads` 之后存进 `self.results`，最后一次性落盘。于是一次数小时的训练结束后，你拿到一份完整档案：数据集多大、SFT 练了几轮、两次评估各多少分。

配合 `config.json`（输入存档）+ `training_results.json`（输出存档）+ 第 40-43 行给每条日志打时间戳的 `log()` 方法，这条流水线满足了**实验可复现**的最低三件套：输入可查、过程可溯、输出可比。跑十组实验之后，你会感谢这三行代码。

---

## 🎯 剥完之后：它们在本章的位置

06 + 07 是本章的"总装与质检车间"：

| 阶段 | 复用的零件 |
|---|---|
| stage1 数据准备 | `01_dataset_loading.py` 的加载协议 |
| stage2/4 训练 | `04_sft_training.py`、`05_grpo_training.py` 的配置 |
| stage3/5 评估 | `07_model_evaluation.py` 的 evaluate action + 02 篇的奖励函数 |
| 全程 | `03_lora_configuration.py` 的 LoRA（`use_lora: True` 贯穿始终） |

学习路径上的建议：先用 00 冒烟、逐个跑 01-05 理解零件，最后跑 06 看零件合体、跑 07 看成绩单。如果说前面的文件教你"每个旋钮是干嘛的"，06 + 07 教的是**怎么把一次训练组织成一次严谨的实验**。
