# 08_distributed_training.py & accelerate_configs 剥洋葱式讲解

> 源文件：`code/chapter11/08_distributed_training.py`（104 行）+ `accelerate_configs/` 下三个 YAML（DDP / ZeRO-2 / ZeRO-3）
> 一句话概括：**训练代码一行不改，换一个 YAML 配置文件就从单卡切到 4 卡 DDP 或 DeepSpeed ZeRO——分布式的复杂度被 Accelerate 全部吸收，你只需要懂"该选哪个 YAML"。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

单卡训练撞到两堵墙：**慢**（数据太多）和**装不下**（模型太大）。分布式训练分别用"数据并行"和"模型分片"来拆墙。传统印象里分布式代码要写进程组初始化、梯度同步、数据切分……而这个文件想证明的恰恰相反——看它的使用说明（第 7-18 行）：

```
1. 单GPU:          python 07_distributed_training.py
2. 多GPU DDP:      accelerate launch --config_file accelerate_configs/multi_gpu_ddp.yaml ...
3. DeepSpeed ZeRO-2: accelerate launch --config_file accelerate_configs/deepspeed_zero2.yaml ...
4. DeepSpeed ZeRO-3: accelerate launch --config_file accelerate_configs/deepspeed_zero3.yaml ...
```

**四种运行方式，同一份 Python 脚本**。区别只在启动命令带哪个 YAML。第 81-83 行的注释点破了机制：训练代码里的 `rl_tool.run(config)` 原封不动，Accelerate 在启动时拉起多进程、注入环境变量，框架内部据此自动做梯度同步和数据切分。

---

## 🧅 第二层：多进程世界的生存法则（第 36-46、69-78 行）

```python
world_size = int(os.environ.get("WORLD_SIZE", 1))
local_rank = int(os.environ.get("LOCAL_RANK", 0))
```

`accelerate launch` 之后，**这份脚本会被 4 个进程同时执行**（每 GPU 一个）。两个环境变量是进程的"身份证"：`WORLD_SIZE` 是总进程数，`LOCAL_RANK` 是"我是第几号"。单卡直跑时环境变量不存在，`get(..., 1)` 和 `get(..., 0)` 的默认值让脚本优雅退化——这就是"同一份脚本四种跑法"的实现细节。

由此产生分布式脚本的第一条纪律（第 69、86 行）：

```python
if local_rank == 0:
    print("\n训练配置:")   # 只在主进程打印
```

不加这个判断，所有日志都会被打印 4 遍。打印只是小事，推而广之——**写文件、存模型、上报指标，都只应由 0 号进程做**，否则轻则日志混乱，重则多进程同时写同一个文件把 checkpoint 写坏。

第 53-54 行的注释则是分布式调参第一课：

```python
# 总batch size = batch_size × num_gpus × gradient_accumulation_steps
```

配置里写 `batch_size: 2`，4 卡 + 梯度累积 4 步（见 ZeRO YAML），实际有效 batch 是 2×4×4=32。忘了这笔账，多卡训练的学习率就会失配。

---

## 🧅 第三层：三个 YAML 的阶梯——从复制到分片

**`multi_gpu_ddp.yaml`（DDP，数据并行）**：

```yaml
distributed_type: MULTI_GPU
num_processes: 4
mixed_precision: fp16
```

每个 GPU 持有**完整的模型副本**，各吃各的数据分片，反向传播后 all-reduce 平均梯度。速度最快（通信量只有梯度），但显存一点不省——模型装不进单卡时 DDP 无能为力。

要看懂 ZeRO 省的是什么，先算一笔账：混合精度 + Adam 下，每个参数要占 **16 字节** = fp16 权重 2 + fp16 梯度 2 + 优化器状态 12（fp32 主权重 4 + 动量 4 + 方差 4）。**优化器状态占了 3/4**——它就是 ZeRO 逐级开刀的对象：

| 级别 | 分片内容 | 单卡持有 | 显存节省 | 代价 |
|---|---|---|---|---|
| DDP | 不分片 | 16 字节/参数 | 0 | — |
| ZeRO-1 | 优化器状态 | 2+2+12/N | 中 | 少量通信 |
| ZeRO-2 | + 梯度 | 2+(2+12)/N | ~30% | 通信略增 |
| ZeRO-3 | + 模型参数 | 16/N | ~50%+ | 前向时要临时聚合参数，通信开销最大 |

对照两个 DeepSpeed YAML 的差异行：

```yaml
# deepspeed_zero2.yaml            # deepspeed_zero3.yaml
zero_stage: 2                     zero_stage: 3
offload_optimizer_device: none    offload_optimizer_device: cpu  # 优化器状态卸载到CPU
offload_param_device: none        offload_param_device: cpu      # 参数卸载到CPU
zero3_init_flag: false            zero3_init_flag: true
```

ZeRO-3 还顺手打开了 **CPU offload**——显存实在不够就把分片进一步搬到内存，用 PCIe 传输时间换显存空间。`zero3_init_flag: true` 则让模型在加载那一刻就直接以分片形态初始化，避免"先完整加载再切分"时的显存峰值——没有它，"单卡装不下的模型"连训练第一步都到不了。

选择口诀（来自 `accelerate_configs/README.md`）：**装得下用 DDP 求快；装得下但紧张用 ZeRO-2；装不下用 ZeRO-3**。

---

## 🧅 洋葱心：一行没变的那一行（第 83 行）

```python
result = rl_tool.run(config)
```

整个文件的洋葱心是这行**和单卡版一模一样的调用**。它前面的注释就是本篇的中心思想："训练代码完全不需要修改! Accelerate会自动处理分布式训练的所有细节"。

这背后是一次干净的关注点分离：

- **算法逻辑**（练什么、怎么练）→ Python 代码和 config 字典，01-07 篇已讲完；
- **资源编排**（几张卡、怎么切、精度多少）→ YAML 文件，交给启动器。

第 99 行还留了一句清醒剂：4 卡的理论加速比标为 `~3.4x` 而不是 4x——**梯度同步的通信和数据加载都要交税**，分布式从来不是免费的线性扩展。

---

## 🎯 剥完之后：它在本章的位置

08 是本章的"出厂放大器"：00-07 在单卡上把流程调通、把配方（SFT + GRPO + LoRA + 奖励函数）验证正确，08 负责把同一配方原样放大到多卡——放大过程中**唯一变的是启动命令**。

它也补全了本章的显存工具箱全景：**LoRA 减少要训练的参数（03 篇），混合精度压缩每个数的字节数（YAML 里的 fp16），ZeRO 把剩下的开销切片分摊到多卡，offload 再把溢出的部分推到 CPU 内存**——四层手段层层叠加，就是"消费级硬件训练大模型"这件事在工程上的完整答案。
