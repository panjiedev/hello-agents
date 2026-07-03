# ELIZA.py 剥洋葱式讲解

> 源文件：`code/chapter2/ELIZA.py`（84 行）
> 一句话概括：**复刻 1966 年 MIT 的传奇聊天机器人 ELIZA——一个只靠正则匹配和代词替换就让无数人"聊上头"的心理治疗师，符号主义 AI 时代的活化石。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

运行它，你会和一位"心理治疗师"对话：

```
Therapist: Hello! How can I help you today?
You: I am feeling sad
Therapist: How long have you been feeling sad?
You: I need a vacation
Therapist: Why do you need a vacation?
```

看起来它"听懂"了你的话——但整个程序**没有一行代码理解语义**。它只有三样东西：一张"正则模式 → 回复模板"的规则表、一张代词互换表、一个匹配循环。剥开洋葱，你会看到"理解"的幻觉是怎么用 84 行代码变出来的。

这就是原版 ELIZA（Joseph Weizenbaum，MIT，1966）的核心思想。它是历史上第一个引发"人机对话"热潮的程序，也留下了著名的 **ELIZA 效应**：人类会不由自主地把理解力投射到机器身上——哪怕机器只是在做字符串替换。

---

## 🧅 第二层：`rules` 规则库——机器的全部"知识"（第 5-41 行）

```python
rules = {
    r'I need (.*)': [
        "Why do you need {0}?",
        "Would it really help you to get {0}?", ...
    ],
    r'I am (.*)': [
        "Did you come to me because you are {0}?", ...
    ],
    r'.* mother .*': [
        "Tell me more about your mother.", ...
    ],
    r'.*': [                                # 万能兜底
        "Please tell me more.", ...
    ]
}
```

三个设计值得剥开：

1. **模式即知识**。每条正则代表一类"值得接话的句式"：`I need (.*)` 里的 `(.*)` 是**捕获组**，把用户需要的东西原样抓下来，回填进模板的 `{0}` 插槽。机器根本不知道 "vacation" 是什么，但把它塞回 "Why do you need ___?" 就显得懂了。
2. **顺序即优先级**。Python 3.7+ 的字典保持插入顺序，`respond` 按序遍历——具体模式在前，万能的 `r'.*'` 垫底。**兜底规则保证永远有话可接**（"Please tell me more."），冷场是聊天机器人的死刑。
3. **每个模式配多个模板，随机挑一个**（`random.choice`）。同样问 "I need X" 三次会得到三种反问——一点点随机性就能大幅延缓"它在复读"的穿帮时刻。

另外注意 `mother`/`father` 两条规则**没有捕获组**——它们不需要复述你的话，只要检测到关键词就抛出预设的追问（"Tell me more about your mother."）。经典的心理治疗师话术，也是最偷懒的规则：**关键词触发，内容全靠编**。

---

## 🧅 第三层：`respond`——四步流水线（第 59-74 行）

```python
def respond(user_input):
    for pattern, responses in rules.items():
        match = re.search(pattern, user_input, re.IGNORECASE)   # ① 逐条试匹配
        if match:
            captured_group = match.group(1) if match.groups() else ''  # ② 抓内容
            swapped_group = swap_pronouns(captured_group)        # ③ 换人称
            response = random.choice(responses).format(swapped_group)  # ④ 填模板
            return response
```

- `re.IGNORECASE` 让 "i need" 和 "I NEED" 一视同仁；
- `match.group(1) if match.groups() else ''`——防御 mother/father 这类无捕获组的模式：没有组就填空字符串（模板里也没有 `{0}`，填了也不显示）；
- 第一个命中的规则立即 `return`——**先到先得**，再次体现顺序即优先级。

一个有趣的细节：第 74 行的 `return random.choice(rules[r'.*'])` 实际上是**永远执行不到的死代码**——因为 `r'.*'` 在循环里就能匹配任何输入（包括空串），循环内必然 return。它是一道"以防万一"的保险，也是一个小小的代码考古乐趣。

---

## 🧅 洋葱心：`swap_pronouns`——幻觉的发动机（第 44-57 行）

```python
pronoun_swap = {
    "i": "you", "you": "i", "me": "you", "my": "your",
    "am": "are", "was": "were", "i'd": "you would", ...
}

def swap_pronouns(phrase):
    words = phrase.lower().split()
    swapped_words = [pronoun_swap.get(word, word) for word in words]
    return " ".join(swapped_words)
```

为什么说这 6 行是洋葱心？看看没有它会发生什么：

```
你说:   I am afraid of my boss
抓到:   "afraid of my boss"
不换人称: "How long have you been afraid of MY boss?"   ← 穿帮！瞬间出戏
换了人称: "How long have you been afraid of YOUR boss?" ← 像真的在听
```

把第一人称翻转成第二人称（i→you、my→your、am→are…），机器复述你的话时视角才是对的。**"它在认真听我说话"的幻觉，一半功劳属于这张小小的替换表。**实现上就是查字典：`pronoun_swap.get(word, word)` ——在表里就换，不在就原样保留。

它当然也粗糙得可爱：逐词替换不看语境，"you" 一律变 "i"，遇到复杂从句就会露馅——但在"心理治疗师"这个精心挑选的场景里（治疗师本来就该少说多问、把话题抛回来），破绽被话术完美掩护。**Weizenbaum 真正的天才之处是场景选择，而不是算法。**

---

## 🎯 剥完之后：ELIZA 与 LLM 隔着什么？

| | ELIZA（1966） | LLM 智能体（chapter1） |
|---|---|---|
| "知识"来源 | 程序员手写的 7 条正则 | 万亿 token 语料中学到的参数 |
| 应对没见过的输入 | 落入 `.*` 兜底，装傻 | 泛化生成 |
| 有没有"理解" | 纯字符串操作，零理解 | 有争议，但至少有可用的语义表示 |
| 骨架 | **匹配 → 抽取 → 变换 → 填模板** | 惊人地相似：解析 → 调工具 → 填回历史 |

最后一行值得多看一眼：chapter1 的 `FirstAgentTest.py` 用正则解析模型输出、查表调用工具、把结果填回模板化的历史——**结构上和 ELIZA 是同一族**。变的是中间那颗"脑"：从 7 条手写规则换成了千亿参数的 Transformer（chapter3），于是同样的骨架突然真的能干活了。

这就是本章把 ELIZA 放在全书开头的用意：**智能体的躯壳六十年没变，变的是灵魂。**
