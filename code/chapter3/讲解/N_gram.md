# N_gram.py 剥洋葱式讲解

> 源文件：`code/chapter3/N_gram.py`（30 行）
> 一句话概括：**用最朴素的"数数"方法，算出一句话出现的概率——这就是大语言模型最古老的祖先：N-gram 语言模型。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

语言模型的本质问题是：**"这句话像不像人话？"** 用数学语言说，就是给一个句子算一个概率 P(句子)。

这个脚本用一个只有 6 个词的迷你语料库，手工计算出句子 `"datawhale agent learns"` 的概率。运行它，你会看到四行打印，分别对应概率计算的四个步骤，最终得到 P ≈ 0.5。

不需要神经网络，不需要 GPU，只需要**数数和除法**。

---

## 🧅 第二层：核心思想——链式法则 + 马尔可夫假设

剥开一层，看它背后的数学。一个句子的概率按**链式法则**展开：

```
P(datawhale agent learns)
  = P(datawhale) × P(agent | datawhale) × P(learns | datawhale agent)
```

问题：`P(learns | datawhale agent)` 依赖前面**所有**词，语料一大根本统计不过来。

于是引入 **马尔可夫假设（Bigram / 2-gram）**：每个词只依赖它的**前一个词**：

```
P(learns | datawhale agent)  ≈  P(learns | agent)
```

这就是文件名 N_gram 的含义——N=2 时叫 bigram，本脚本实现的正是它。

---

## 🧅 第三层：程序流程——四个步骤逐行对应

### 准备工作（第 4-6 行）

```python
corpus = "datawhale agent learns datawhale agent works"
tokens = corpus.split()      # 切成 6 个词
total_tokens = len(tokens)   # 6
```

语料库只有一句话、6 个词元（token）。`split()` 就是最简陋的分词器。

### 第一步：P(datawhale)（第 9-11 行）

```python
count_datawhale = tokens.count('datawhale')   # 出现 2 次
p_datawhale = count_datawhale / total_tokens  # 2/6 ≈ 0.333
```

句子开头词的概率 = 它在语料中的出现次数 ÷ 总词数。**概率就是频率**，这是整个 N-gram 模型的灵魂。

### 第二步：P(agent | datawhale)（第 15-20 行）

```python
bigrams = zip(tokens, tokens[1:])
bigram_counts = collections.Counter(bigrams)
```

这两行是全文最巧妙的地方，值得单独剥开（见第四层）。

```python
p_agent_given_datawhale = count_datawhale_agent / count_datawhale  # 2/2 = 1.0
```

条件概率 = "datawhale agent" 连着出现的次数 ÷ "datawhale" 出现的次数。语料里 datawhale 后面 100% 跟着 agent，所以是 1.0。

### 第三步：P(learns | agent)（第 23-26 行）

同样的套路：`("agent", "learns")` 出现 1 次，`agent` 出现 2 次（另一次后面跟的是 works），所以概率为 1/2 = 0.5。

### 最后：连乘（第 29-30 行）

```python
p_sentence = 0.333 × 1.0 × 0.5 ≈ 0.167
```

三个概率相乘，得到整句话的概率。

---

## 🧅 第四层（洋葱心）：`zip(tokens, tokens[1:])` 这一行

```python
bigrams = zip(tokens, tokens[1:])
```

这是用一行 Python 生成所有"相邻词对"的经典写法：

```
tokens      = [datawhale, agent, learns, datawhale, agent, works]
tokens[1:]  = [agent, learns, datawhale, agent, works]
zip 结果    = (datawhale,agent) (agent,learns) (learns,datawhale) (datawhale,agent) (agent,works)
```

把序列和"错开一位的自己"配对，就得到了所有 bigram。再交给 `collections.Counter` 统计，`bigram_counts[('datawhale', 'agent')]` 就能直接查出"datawhale 后面跟 agent"出现了几次。

**注意一个细节**：`zip` 返回的是一次性迭代器，被 `Counter` 消费后就空了——所以代码把 `Counter` 的结果存下来反复查询，而不是反复 `zip`。

---

## 🎯 剥完之后：它的意义与局限

- **意义**：N-gram 揭示了语言模型的本质——**预测下一个词**。今天的 GPT、Qwen 做的仍然是同一件事，只是把"数数"换成了千亿参数的神经网络。
- **局限一（数据稀疏）**：只要语料里没出现过 "agent sleeps"，它的概率就是 0，整句概率连乘后也是 0。
- **局限二（视野狭窄）**：bigram 只看前一个词，无法理解长距离依赖。

这两个局限，正是后面 `Word_Embedding.py`（用连续向量代替离散计数）和 `Transformer.py`（用注意力机制看全句）要解决的问题。
