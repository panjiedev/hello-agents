# Chapter 3 代码剥洋葱讲解

对 `code/chapter3/` 下五个代码文件的逐层拆解。每篇文档都采用"剥洋葱"结构：从最外层的**这个文件在干什么**开始，一层层剥到最核心的**那几行关键代码**，最后回到全局，说明它在大模型技术演进中的位置。

## 推荐阅读顺序

按语言模型技术的演进脉络阅读，前后文档互有呼应：

| 顺序 | 文档 | 源文件 | 主题 | 解决的问题 |
|---|---|---|---|---|
| 1 | [N_gram.md](N_gram.md) | `N_gram.py` | N-gram 统计语言模型 | 语言模型的本质：给句子算概率、预测下一个词 |
| 2 | [Word_Embedding.md](Word_Embedding.md) | `Word_Embedding.py` | 词向量与余弦相似度 | 让词义变成可计算的向量（king − man + woman ≈ queen） |
| 3 | [BPE.md](BPE.md) | `BPE.py` | BPE 子词分词算法 | 词表怎么定：在"整词"和"字符"之间找平衡 |
| 4 | [Transformer.md](Transformer.md) | `Transformer.py` | 完整 Transformer 架构 | 一词多义与长距离依赖：注意力机制登场 |
| 5 | [Qwen.md](Qwen.md) | `Qwen.py` | 调用真实大模型推理 | 前四个原理的工业级会师：跑通 Qwen 的完整推理链路 |

## 一条主线

五个文件讲的是同一个故事的五个章节：

> **预测下一个词**（N-gram）→ 词义要能计算（Word Embedding）→ 词元要切得聪明（BPE）→ 用注意力理解上下文（Transformer）→ 组装成真实可用的大模型（Qwen）。
