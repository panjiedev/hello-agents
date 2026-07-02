# 00_quick_test.py & 01_dataset_loading.py 剥洋葱式讲解

> 源文件：`code/chapter11/00_quick_test.py`（146 行）+ `code/chapter11/01_dataset_loading.py`（285 行）
> 一句话概括：**用 `RLTrainingTool` 的统一入口跑通"数据加载 → SFT → GRPO → 奖励函数"的全流程冒烟测试，并剥开训练数据的两副面孔——SFT 格式与 RL 格式。**

---

## 🧅 第一层（最外层）：这两个文件在干什么？

前面章节的智能体都是"拿现成模型用"，本章第一次要**训练模型本身**。训练最怕的是什么？跑了三小时才发现配置写错了。所以本章第一个文件是 `00_quick_test.py`：用 0.6B 小模型 + 10 个样本 + 1 轮训练，两三分钟内把整条流水线走一遍——像发射火箭前的点火测试。

`01_dataset_loading.py` 则专注流水线的第一节车厢：**数据**。它用 6 个示例函数演示怎么加载 GSM8K（小学数学应用题数据集），以及同一份数据如何被格式化成 SFT 和 RL 两种截然不同的形态。

两个文件的所有操作都长一个样：

```python
tool = RLTrainingTool()
result = tool.run({"action": "...", ...})   # 传配置字典
data = json.loads(result)                   # 拿 JSON 结果
```

这就是本章的**总协议**：训练框架的全部能力（加载数据、训练、评估、造奖励函数）都收敛到一个 `tool.run(config)` 接口，配置字典里的 `action` 字段决定干什么。

---

## 🧅 第二层：为什么训练工具长得像"智能体工具"？（00 第 15、28 行）

```python
from hello_agents.tools import RLTrainingTool
...
tool = RLTrainingTool()
```

注意导入路径是 `hello_agents.tools`——训练能力被封装成了和 chapter4 搜索工具同款的 **Tool 接口**（吃 JSON 配置、吐 JSON 结果）。这不是偶然：本章的主题是 *Agentic RL*，"训练一个模型"本身也可以是智能体调用的一种工具。副作用也很实用——所有输入输出都是可序列化的 JSON，天然适合记日志、写进 `training_results.json`（见 06 篇）。

`00_quick_test.py` 依次跑四个 action（第 40-124 行）：

| 测试 | action | 关键配置 |
|---|---|---|
| 1 数据加载 | `load_dataset` | `max_samples: 5` |
| 2 SFT 训练 | `train` + `algorithm: "sft"` | 10 样本、1 轮、LoRA r=8 |
| 3 GRPO 训练 | `train` + `algorithm: "grpo"` | 同上 |
| 4 奖励函数 | `create_reward` | `reward_type: "accuracy"` |

对比第 59-70 行和第 87-98 行会发现：SFT 和 GRPO 的配置**只差 `algorithm` 一个字段**。算法切换的复杂度全部被框架吞掉了——这正是配置驱动设计的价值。

---

## 🧅 第三层：一份数学题，两副面孔（01 第 21-88 行）

同一道 GSM8K 题目，喂给 SFT 和喂给 RL，格式完全不同。

**SFT 格式**（第 25-30 行的 docstring）：

```python
{
    "prompt": "Question: ...\n\nLet's solve this step by step:\n",
    "completion": "Step 1: ...\nFinal Answer: 42",
    "text": "Question: ... Step 1: ...\nFinal Answer: 42"
}
```

**RL 格式**（第 61-67 行的 docstring）：

```python
{
    "prompt": "<|im_start|>user\nQuestion: ...\n<|im_end|>\n<|im_start|>assistant\n",
    "ground_truth": "42",
    ...
}
```

三个耐人寻味的差异：

1. **SFT 有 `completion`（完整解答），RL 只有 `ground_truth`（一个数字 42）**。SFT 是"抄答案"——模型逐 token 模仿人写的解题过程；RL 是"只判对错"——模型自己写解答，奖励函数只拿最终答案对账。这一个字段的差异就是两种训练范式的分水岭。
2. **RL 的 prompt 带 `<|im_start|>` 聊天模板**，所以加载 RL 格式需要额外传 `model_name`（第 76 行）——模板是分词器决定的，不同模型的模板不同。RL 训练时模型要真的"开口生成"，prompt 必须和推理时一模一样。
3. **RL 的 prompt 结尾停在 `assistant\n`**——刚好是模型该接话的位置，后面的一切由模型自己采样出来。

---

## 🧅 洋葱心：`max_samples` 这一个参数（01 第 148 行）

```python
"max_samples": None  # None = 使用全部数据
```

整个 01 文件 6 个示例，真正在变的只有 `max_samples`（5 → 100 → None）和 `split`（train/test）。这个不起眼的参数是本章反复出现的**实验节流阀**：

- `max_samples=5`：验证数据管道通不通（秒级）；
- `max_samples=100`：验证训练收不收敛（分钟级）；
- `max_samples=None`：全量 7500 样本正式训练（小时级）。

第 154-157 行故意把全量加载注释掉了——本章示例代码的一贯风格：**贵的操作默认注释，先让读者零成本读懂结构，再自己拍板花算力**。

---

## 🎯 剥完之后：它们在本章的位置

这两个文件是本章的"玄关"：

- `00_quick_test.py` 给出全章地图——后面 02～07 每个文件都是把它四个测试中的某一个拿出来放大细讲；
- `01_dataset_loading.py` 定义了数据契约——SFT 格式流向 `04_sft_training.py`，RL 格式流向 `05_grpo_training.py`，`ground_truth` 字段则是 `02_reward_functions.py` 里奖励计算的对账依据。

先跑 00 确认环境没问题，再往下剥。
