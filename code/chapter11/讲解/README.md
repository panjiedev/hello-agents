# Chapter 11 代码剥洋葱讲解

对 `code/chapter11/` 下九个代码文件（含 `accelerate_configs/` 配置目录）的逐层拆解。每篇文档都采用"剥洋葱"结构：从最外层的**这个文件在干什么**开始，一层层剥到最核心的**那几行关键代码**，最后回到全局，说明它在模型训练流水线中的位置。

## 推荐阅读顺序

先冒烟测试摸清全貌，再依次剥开四个核心零件（数据、奖励、LoRA、两种训练），最后看总装、验收与放大：

| 顺序 | 文档 | 源文件 | 主题 | 一句话 |
|---|---|---|---|---|
| 1 | [dataset_loading.md](dataset_loading.md) | `00_quick_test.py`、`01_dataset_loading.py` | 冒烟测试与数据格式 | 一个 `tool.run(config)` 协议贯穿全章；同一道数学题的 SFT / RL 两副面孔 |
| 2 | [reward_functions.md](reward_functions.md) | `02_reward_functions.py` | 奖励函数设计 | RL 的"评分老师"：准确性打底，长度惩罚/步骤奖励装饰器式套娃组合 |
| 3 | [lora_configuration.md](lora_configuration.md) | `03_lora_configuration.py` | LoRA 参数高效微调 | 冻结基座，只在 q_proj/v_proj 旁挂低秩矩阵，显存 -70%、模型文件 10MB |
| 4 | [sft_training.md](sft_training.md) | `04_sft_training.py` | SFT 监督微调 | 逐 token 模仿人写的解答：学会格式与套路，为 GRPO 冷启动 |
| 5 | [grpo_training.md](grpo_training.md) | `05_grpo_training.py` | GRPO 强化学习 | 一题生成一组解答，组内比较定奖惩——无 critic 的轻量 RL |
| 6 | [pipeline_and_evaluation.md](pipeline_and_evaluation.md) | `06_complete_pipeline.py`、`07_model_evaluation.py` | 端到端管线与评估 | 六阶段自动流水线 + "基线 < SFT < GRPO"三方验收 |
| 7 | [distributed_training.md](distributed_training.md) | `08_distributed_training.py`、`accelerate_configs/` | 分布式训练 | 代码一行不改，换 YAML 就从单卡切到 DDP / ZeRO-2 / ZeRO-3 |

## 一条主线

本章讲的是**智能体模型的后训练——SFT + GRPO 强化学习**。前面章节都在"用"模型，本章第一次"炼"模型，配方是推理模型（DeepSeek-R1 等）验证过的两级火箭：

> **SFT**（临摹字帖：抄人写的解答，学会格式与套路，但不保证做对新题）
> **GRPO**（下场做题：自己生成解答、奖励函数判分、组内排名定奖惩，把正确率真正抬上去）

点火顺序不可颠倒——没有 SFT 钉死 `Final Answer:` 格式，奖励函数提取不到答案，GRPO 的组内比较全是零分，无从"抓起"。**先模仿，再超越。**

贯穿七篇文档的两条工程暗线：

1. **一切皆配置**：从加载数据到 4 卡 ZeRO-3，所有能力都收敛于 `RLTrainingTool.run(config)` 一个 JSON 协议——换算法改一个字段，换硬件改一个 YAML，训练代码从头到尾没变过；
2. **显存工具箱层层叠加**：LoRA 砍可训练参数（篇 3）→ 小 batch / 小秩兜底（篇 4）→ fp16 混合精度压字节 → ZeRO 分片摊到多卡、offload 推到 CPU（篇 7）——"消费级硬件训练大模型"的完整答案。

> 前置阅读：chapter4 的讲解（`code/chapter4/讲解/`）解释了怎么给现成 LLM 装上手脚；本章反过来打磨"大脑"本身——当提示词工程到达上限，就该让权重动起来了。
