# Qwen.py 剥洋葱式讲解

> 源文件：`code/chapter3/Qwen.py`（59 行）
> 一句话概括：**用 Hugging Face `transformers` 库加载真实的开源大模型 Qwen1.5-0.5B-Chat，跑通"输入一句话 → 模型回答"的完整推理链路。**

前面三个文件（N_gram、Word_Embedding、Transformer）都是原理玩具，这个文件是"实弹演习"：本章理论的所有环节——分词、模板、生成、解码——在这 59 行里逐一落地。

---

## 🧅 第一层（最外层）：这个文件在干什么？

运行它，会自动下载一个 5 亿参数的对话模型到本地，然后问它"你好，请介绍你自己"，最后打印模型的回答。全程只用 CPU 也能跑（0.5B 是 Qwen 系列最小的型号，正是为了教学演示能跑得动）。

从外面看，整个脚本就是一条流水线：

```
对话消息 → 套模板 → 编码成数字 → 模型生成 → 裁掉输入 → 解码回文字
```

下面一层层剥开每个环节。

---

## 🧅 第二层：环境与加载（第 1-21 行）

```python
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"
```

**第一行就是国内用户的救命稻草**：Hugging Face 官方源在国内经常连接失败（Connection aborted），这行把下载地址切到国内镜像。注意它必须写在 `import transformers` **之前**——环境变量是在库导入时读取的。

```python
device = "cuda" if torch.cuda.is_available() else "cpu"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id).to(device)
```

三个要点：

- **两件套分开加载**：`tokenizer`（分词器，`BPE.py` 原理的工业级实现）负责"文字 ↔ 数字"的翻译；`model`（`Transformer.py` 中 Decoder 架构的放大版）只处理数字。
- **`AutoXxx` 的 "Auto"**：根据模型仓库里的配置文件自动选择正确的模型类，你不需要知道 Qwen 用的具体是哪个 Python 类。
- **`CausalLM`** = 因果语言模型 = 只能看过去、不能看未来、逐词预测——正是 `Transformer.py` 里 `torch.tril` 下三角掩码所保证的那种模型。

---

## 🧅 第三层：对话模板——被多数人忽略的关键一步（第 24-34 行）

```python
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "你好，请介绍你自己。"}
]
text = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
```

剥开这一步：模型底层**只会补全文本**，根本没有"对话"的概念。所谓聊天，是把消息列表**渲染成一段带特殊标记的纯文本**。对 Qwen 来说，渲染结果是：

```
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
你好，请介绍你自己。<|im_end|>
<|im_start|>assistant
```

两个参数的含义：

- `tokenize=False`：先只要渲染后的字符串，不急着转数字（方便你 print 出来观察）；
- `add_generation_prompt=True`：**在末尾补上 `<|im_start|>assistant`**——相当于把笔递到模型手里说"该你说了"。漏掉它，模型可能会自顾自续写用户的话而不是回答。

每家模型的模板格式都不同（Qwen 用 ChatML，Llama 用另一套），`apply_chat_template` 会自动使用该模型自带的模板——**手工拼接对话字符串是新手最常见的翻车点**，这个 API 就是为了消灭它。

---

## 🧅 第四层：编码与生成（第 37-47 行）

```python
model_inputs = tokenizer([text], return_tensors="pt").to(device)
```

分词器把字符串切成词元并映射为 ID，`return_tensors="pt"` 表示返回 PyTorch 张量，`.to(device)` 把数据搬到和模型相同的设备上（**数据和模型必须同设备，否则当场报错**——初学者高频坑）。打印出来能看到 `input_ids`（词元 ID 序列）和 `attention_mask`（有效位置标记，即 `Transformer.py` 里 padding mask 的来源）。

```python
generated_ids = model.generate(model_inputs.input_ids, max_new_tokens=512)
```

`generate` 是整个脚本的心脏。剥开这一行，它内部是个循环，做的事你在 `Transformer.py` 已经全见过：

```
while 未生成 <|im_end|> 且长度 < max_new_tokens:
    logits = model(当前全部 token)      # 前向传播，得到 (1, seq_len, vocab_size)
    下一个词 = 从最后一个位置的 logits 中选出
    把它拼到序列末尾，继续
```

即**自回归生成**：每次只产出一个词元，再把它喂回去预测下一个。`max_new_tokens=512` 是安全阀，防止模型停不下来。

---

## 🧅 洋葱心：裁掉输入再解码（第 51-56 行）

```python
generated_ids = [
    output_ids[len(input_ids):]
    for input_ids, output_ids in zip(model_inputs.input_ids, generated_ids)
]
response = tokenizer.batch_decode(generated_ids, skip_special_tokens=True)[0]
```

最内核的一个细节，也是最容易写错的地方：**`generate` 返回的序列 = 你的输入 + 新生成的内容**（因为自回归就是在原序列上不断追加）。如果直接解码，回答开头会把 system prompt 和用户问题原样复读一遍。

所以用 `output_ids[len(input_ids):]` 把前面属于输入的部分切掉，只留新生成的词元。`zip` 写法是为了兼容一批多条输入的情况（本例只有 1 条）。

最后 `batch_decode` 把 ID 序列翻译回文字，`skip_special_tokens=True` 顺手删掉 `<|im_end|>` 之类的控制标记，得到干净的回答。

---

## 🎯 剥完之后：本章知识的大集合

这 59 行脚本是 chapter3 四个文件的会师点：

| 脚本中的环节 | 对应的原理文件 |
|---|---|
| `tokenizer` 切词 | `BPE.py`（子词分词算法） |
| `input_ids` → 内部向量 | `Word_Embedding.py`（嵌入查询表） |
| `model` 的前向传播 | `Transformer.py`（Decoder-only 架构 + 因果掩码） |
| `generate` 逐词预测 | `N_gram.py`（同样是"预测下一个词"，只是从数数换成了 5 亿参数） |

理解了这条链路，换任何开源模型（Llama、DeepSeek、更大的 Qwen）都是同一套代码，改一下 `model_id` 而已。
