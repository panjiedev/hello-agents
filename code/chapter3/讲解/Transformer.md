# Transformer.py 剥洋葱式讲解

> 源文件：`code/chapter3/Transformer.py`（248 行）
> 一句话概括：**用约 250 行 PyTorch 从零复现《Attention Is All You Need》（2017）中的完整 Transformer——今天所有大语言模型的共同骨架。**

这个文件是 chapter3 中最有分量的一个，也最适合剥洋葱：它的类层层嵌套，天然就是一颗洋葱。我们从最外皮（怎么用）一路剥到洋葱心（注意力公式的那 4 行）。

```
第一层  __main__ 演示           ——怎么用
第二层  Transformer             ——总装配图 + 掩码
第三层  Encoder / Decoder       ——堆叠 N 层
第四层  EncoderLayer / DecoderLayer ——单层的内部结构
第五层  MultiHeadAttention / FFN / PositionalEncoding ——三大零件
洋葱心  scaled_dot_product_attention ——4 行核心公式
```

---

## 🧅 第一层：`__main__` 演示（第 225-249 行）——只看输入输出

```python
model = Transformer(src_vocab_size=5000, tgt_vocab_size=5000,
                    d_model=512, num_layers=6, num_heads=8,
                    d_ff=2048, dropout=0.1, max_len=100)

src = torch.randint(1, 5000, (2, 10))   # 2 句源语言，每句 10 个词的 ID
tgt = torch.randint(1, 5000, (2, 12))   # 2 句目标语言，每句 12 个词的 ID
output = model(src, tgt)                 # torch.Size([2, 12, 5000])
```

把模型当黑盒：**吃进两串整数（词 ID），吐出形状 `(2, 12, 5000)` 的张量**——对目标句的每个位置，给出词表中 5000 个词各自的得分（logits）。取 softmax 就是"下一个词是谁"的概率分布。

超参数全部沿用原论文的 base 配置：d_model=512（向量宽度）、6 层、8 头、d_ff=2048。原始 Transformer 是为机器翻译设计的，所以有 src（源语言）和 tgt（目标语言）两路输入。

---

## 🧅 第二层：`Transformer` 类（第 195-222 行）——总装配图

剥开黑盒，里面只有三个部件和一个流程：

```python
def forward(self, src, tgt):
    src_mask, tgt_mask = self.generate_mask(src, tgt)      # ① 造掩码
    encoder_output = self.encoder(src, src_mask)           # ② 编码器读懂源句
    decoder_output = self.decoder(tgt, encoder_output, ...) # ③ 解码器边看边写
    output = self.final_linear(decoder_output)             # ④ 投影到词表
    return output
```

`final_linear` 是 `nn.Linear(512, 5000)`：把每个位置的 512 维语义向量翻译成 5000 个词的得分。

### 这一层的重点：`generate_mask`（第 202-213 行）

掩码回答一个问题：**"每个位置允许看哪些位置？"** 有两种：

**① 填充掩码（padding mask）**——`(src != 0)`：ID 为 0 的位置是补齐用的空白（padding），不许任何人注意它。

**② 未来掩码（causal mask）**——只给解码器：

```python
tgt_sub_mask = torch.tril(torch.ones((tgt_len, tgt_len)))  # 下三角矩阵
```

`torch.tril` 生成下三角矩阵：第 i 行只有前 i 列是 1。含义是**第 i 个词只能看见第 1~i 个词，不能偷看未来**——否则训练时模型直接抄答案，学不到任何东西。这就是"自回归"生成的数学保证，也是 GPT 系列名字里 "causal LM" 的由来。

两种掩码用 `&` 合并：既不许看 padding，也不许看未来。

> 形状细节：`unsqueeze(1).unsqueeze(2)` 把掩码补成 4 维 `(batch, 1, 1, len)`，是为了后面能和注意力得分 `(batch, num_heads, len, len)` 做广播。

---

## 🧅 第三层：`Encoder` / `Decoder`（第 165-193 行）——同一块积木堆 6 次

两个类结构几乎一样，以 Encoder 为例：

```python
def forward(self, x, mask):
    x = self.embedding(x)       # 词 ID → 512 维向量（可训练的查询表）
    x = self.pos_encoder(x)     # 注入位置信息
    for layer in self.layers:   # 同构的 6 层依次加工
        x = layer(x, mask)
    return self.norm(x)
```

要点只有两个：

1. **入口两件套**：`nn.Embedding`（就是 `Word_Embedding.py` 里那个字典的可训练版）+ 位置编码（见第五层）。
2. **`nn.ModuleList` 堆 6 层**：每层结构相同、参数独立，逐层精炼语义。张量形状进出每层都保持 `(batch, seq_len, 512)` 不变——正因如此才能任意堆叠，大模型"堆层数"扩容的可行性根源在此。

Decoder 唯一的区别：每层多接收 `encoder_output`，并且同时用两种掩码。

---

## 🧅 第四层：`EncoderLayer` / `DecoderLayer`（第 113-163 行）——单层解剖

### EncoderLayer：两个子层

```python
def forward(self, x, mask):
    attn_output = self.self_attn(x, x, x, mask)        # 子层1：自注意力
    x = self.norm1(x + self.dropout(attn_output))      # 残差 + LayerNorm
    ff_output = self.feed_forward(x)                   # 子层2：前馈网络
    x = self.norm2(x + self.dropout(ff_output))        # 残差 + LayerNorm
    return x
```

注意 `self_attn(x, x, x, mask)`——**Q、K、V 三个参数传的都是 x 自己**，这就是"自"注意力：句子内部互相打量，交换信息。

每个子层都套着同一个模式：**`norm(x + dropout(子层(x)))`**，拆开是两件事：

- **残差连接 `x + ...`**：给梯度留一条直通深层的高速公路，6 层、60 层都能训得动（借鉴 ResNet）；
- **LayerNorm**：把每个位置的 512 维向量归一化，稳定数值分布。

### DecoderLayer：三个子层（第 150-163 行）

```python
attn_output = self.self_attn(x, x, x, tgt_mask)                          # ① 掩码自注意力
cross_attn_output = self.cross_attn(x, encoder_output, encoder_output, src_mask)  # ② 交叉注意力
ff_output = self.feed_forward(x)                                          # ③ 前馈网络
```

看第 ② 步的传参——这是全文件信息量最大的一行：**Q 来自解码器自己（x），K 和 V 来自编码器输出**。翻译成人话："我（目标句的当前位置）拿着问题（Q），去源句的编码结果（K、V）里查资料"。源语言的信息就是从这唯一的一扇门流进解码器的。

---

## 🧅 第五层：三大零件

### ① `MultiHeadAttention`（第 6-63 行）——多头的"分与合"

```python
def forward(self, Q, K, V, mask=None):
    Q = self.split_heads(self.W_q(Q))   # 线性变换 + 切成 8 个头
    K = self.split_heads(self.W_k(K))
    V = self.split_heads(self.W_v(V))
    attn_output = self.scaled_dot_product_attention(Q, K, V, mask)
    output = self.W_o(self.combine_heads(attn_output))  # 拼回来 + 再变换
    return output
```

**为什么要多头？** 一次注意力只能学一种"关注模式"。把 512 维切成 8 份、每份 64 维（`d_k = d_model // num_heads`），让 8 个头**并行地各学各的**——有的头盯语法搭配，有的头盯指代关系——最后拼接起来过 `W_o` 融合。相当于 8 个各有所长的评审同时读一句话。

`split_heads` 的实现值得看一眼：

```python
x.view(batch_size, seq_length, self.num_heads, self.d_k).transpose(1, 2)
# (batch, seq, 512) → (batch, seq, 8, 64) → (batch, 8, seq, 64)
```

没有复制任何数据，纯靠 `view` 重解释形状 + `transpose` 换轴，就把"1 个 512 维注意力"变成"8 个 64 维注意力"。`combine_heads` 做严格的逆操作（其中 `.contiguous()` 是因为 transpose 后内存不连续，`view` 之前必须整理）。

### ② `PositionWiseFeedForward`（第 65-83 行）——逐位置的小 MLP

```python
x = self.linear2(self.dropout(self.relu(self.linear1(x))))
# 512 → 2048 → ReLU → 512
```

"Position-wise" 的意思：**每个位置独立过同一个两层全连接**，位置之间互不通信。分工由此清晰——**注意力负责"位置间搬运信息"，FFN 负责"每个位置内部深加工"**。先扩到 4 倍（2048）再压回，给非线性变换留足空间；大模型的参数大头其实在这里。

### ③ `PositionalEncoding`（第 85-111 行）——给词向量盖"位置戳"

注意力有个致命盲区：它对所有位置一视同仁，**天生不知道词的顺序**（把句子打乱，注意力的输出只是跟着重排）。解决办法是把位置信息直接加到词向量上：

```python
pe[:, 0::2] = torch.sin(position * div_term)   # 偶数维用 sin
pe[:, 1::2] = torch.cos(position * div_term)   # 奇数维用 cos
```

每个位置得到一个独一无二的 512 维"波形签名"：低维度波长短（区分相邻位置），高维度波长长（编码远距离）——像时钟的秒针、分针、时针共同指出一个时刻。选三角函数还有个数学彩蛋：任意两个位置的编码之间是线性变换关系，模型容易学会"相对位置"的概念。

两个工程细节：

- `div_term` 用 `exp(−log(10000)·2i/d)` 计算而不是直接算幂，纯粹为了数值稳定；
- `register_buffer('pe', ...)`：pe 是**算出来的常量，不是参数**——不参与梯度更新，但会跟着 `model.to(device)` 一起搬到 GPU，也会存进模型文件。这是 PyTorch 中"非参数状态"的标准归宿。

---

## 🧅 洋葱心：`scaled_dot_product_attention`（第 24-38 行）

剥到最后，整个 Transformer 的灵魂只有 4 行：

```python
attn_scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)  # ①
attn_scores = attn_scores.masked_fill(mask == 0, -1e9)                    # ②
attn_probs = torch.softmax(attn_scores, dim=-1)                           # ③
output = torch.matmul(attn_probs, V)                                      # ④
```

即论文中的公式：**Attention(Q,K,V) = softmax(QKᵀ/√d_k)·V**。用图书馆检索来读：

| 行 | 数学 | 直觉 |
|---|---|---|
| ① | QKᵀ/√d_k | 我的**查询**（Q）和每本书的**标签**（K）做点积——匹配度打分 |
| ② | mask → −10⁹ | 禁看的位置（padding/未来）得分打成负无穷 |
| ③ | softmax | 把得分变成总和为 1 的**关注度分配** |
| ④ | ·V | 按关注度加权，把各本书的**内容**（V）混合成答案 |

两个容易剥漏的细节：

- **为什么除以 √d_k？** 两个 64 维随机向量的点积，方差约为 64。不缩放的话得分动辄几十，softmax 会输出接近 one-hot 的极端分布，梯度几乎为零，训练直接停滞。除以 √64=8 把方差拉回 1 左右。这就是名字里 "scaled"（缩放）的含义。
- **为什么是 −1e9 而不是 0？** 掩码要在 softmax **之前**生效。若把得分置 0，softmax 后仍有可观的概率（e⁰=1）；置 −10⁹ 后 e^(−10⁹) ≈ 0，被掩位置的关注度才真正归零。

---

## 🎯 剥完之后：从这里到 GPT / Qwen

把洋葱重新装回去，整个前向传播就是一句话：

> **词 ID → 查表变向量 → 盖位置戳 → 6 层（自注意力交换信息 + FFN 深加工，全程残差护航）→ 线性层投影 → 下一个词的概率分布。**

和现代大模型的对应关系：

- **GPT / Qwen = 只留这份代码的 Decoder 一半**（去掉 Encoder 和交叉注意力），保留掩码自注意力 + FFN，堆几十层、把 d_model 加宽到几千——本质没有变；
- 现代改进主要是零件升级：LayerNorm 挪到子层之前（Pre-Norm）、正弦位置编码换成 RoPE 旋转位置编码、ReLU 换成 SwiGLU——骨架仍是这 248 行。

读懂了这个文件，下一个文件 `Qwen.py` 里那句 `model.generate(...)` 对你就不再是魔法了。
