# HelloAgent_cozeCase.zip 剥洋葱式讲解

> 源文件：`code/chapter5/HelloAgent_cozeCase.zip`（解压后核心是 `workflow/AI_news-draft.yaml`，1457 行）
> 一句话概括：**一个 Coze Chatflow"AI 日报生成器"——六路信息源并行抓取，汇入一个 LLM 节点整编成日报，是本章四个案例里最纯粹的"扇出-扇入"工作流。**

---

## 🧅 第一层（最外层）：这个文件在干什么？

它是从 Coze（扣子）平台导出的一个完整应用包。zip 解压后只有两个文件：

```
Chatflow-AI_news-draft-9132/
├── MANIFEST.yml           # 清单：我是谁（211 字节）
└── workflow/
    └── AI_news-draft.yaml # 工作流本体（56 KB）
```

`MANIFEST.yml` 一共 11 行，说清了三件事：

```yaml
type: Workflow
main:
    name: AI_news
    desc: 用于获取每日AI新闻
    flowMode: 3
```

这个应用的功能：用户随便说一句话触发，它就去 arXiv、GitHub、四家科技媒体的 RSS 抓取当天的 AI 资讯，然后让 DeepSeek 整理成一份带 emoji、带链接的"AI 日报"。

值得注意的是：**这个工作流里没有分支、没有循环、没有记忆——它是四个案例中拓扑最简单的一个**，因此最适合第一个剥。

---

## 🧅 第二层：九个节点的全家福（`AI_news-draft.yaml` 第 7-1443 行）

`nodes:` 数组里一共九个节点，按角色分三类：

| 节点 id | type | 名称 | 干什么 |
|---|---|---|---|
| 100001 | start | 开始 | 接收 `USER_INPUT`（第 8 行起） |
| 174450 | plugin | search | 在 arXiv 搜 "AI" 论文，取 5 篇（第 659 行起） |
| 199381 | plugin | searchRepositories | 在 GitHub 搜 "AI" 仓库，按 updated 排序取 10 个（第 764 行起） |
| 162396 | plugin | rss_reader_36k | 抓 36 氪 RSS（第 1163 行起） |
| 1970248 | plugin | rss_reader_huxiu | 抓虎嗅 RSS |
| 1314359 | plugin | rss_reader_ithome | 抓 IT 之家 RSS |
| 1745449 | plugin | rss_reader_infoq | 抓 InfoQ RSS |
| 134238 | llm | 大模型 | 把六路结果编成日报（第 49 行起） |
| 900001 | end | 结束 | 流式输出日报（第 26 行起） |

六个 plugin 节点全是"零代码调用现成插件"：以 arXiv 节点为例（第 667-709 行），`apiParam` 声明用哪个插件（`pluginName: "arXiv"`），`node_inputs` 写死了查询参数：

```yaml
node_inputs:
    - name: count
      input: { type: integer, value: 5 }
    - name: search_query
      input: { type: string, value: "AI" }
```

四个 RSS 节点更简单，同一个"RSS解析器"插件复用四次，只换一个 `rss_url`（如 36 氪的 `https://www.36kr.com/feed`，第 1163 行起）。

---

## 🧅 第三层：边——扇出与扇入（第 1445-1471 行）

文件末尾的 `edges:` 只有 13 条，画出来是一张漂亮的菱形：

```
                  ┌→ arXiv search ────────┐
                  ├→ GitHub search ───────┤
   开始(100001) ──┼→ rss_36kr ────────────┼──→ 大模型(134238) ──→ 结束(900001)
                  ├→ rss_huxiu ───────────┤
                  ├→ rss_ithome ──────────┤
                  └→ rss_infoq ───────────┘
```

`开始` 一口气连出六条边（第 1446-1457 行）——**六路抓取是并行的**，不需要谁等谁；六个插件的输出又全部指向同一个 LLM 节点（第 1460-1471 行）。扇出提速、扇入汇总，这个形状是数据聚合类工作流的标准答案。

---

## 🧅 第四层：LLM 节点如何"知道"上游数据在哪？（第 172-649 行）

Coze 的数据流不是隐式传递的。LLM 节点的 `node_inputs` 里，**每一个变量都显式声明了来源节点**：

```yaml
- name: articles
  input:
    type: list
    value:
        path: articles        # 取上游输出里的 articles 字段
        ref_node: "162396"    # 上游是 36 氪 RSS 节点
```

同样的写法重复六次：`articles/articles1/articles2/articles3` 对应四路 RSS，`arxiv` 引用 174450，`GitHub` 引用 199381。声明之后，这些名字就能在提示词里当 `{{变量}}` 用了。

另一个细节：导出文件里 GitHub 变量的 schema 占了近 400 行（第 261 行起，`forks_url`、`stargazers_url`……九十多个字段全列出来了）。**低代码平台的导出文件之所以大，大头不是逻辑，而是这些类型声明**——机器要靠它们做连线时的类型检查。

---

## 🧅 洋葱心：藏在 llmParam 里的两段提示词（第 117-150 行）

整个工作流的"智能"浓缩在 LLM 节点的两个字段里。`systemPrompt`（第 136-150 行）定人设与格式：

```
# 角色
你是一位资深且权威的科技媒体编辑...
### 日报输出格式
1. 日报开头显著标注"AI日报"、"by@jasonhuang"和当天时间。
2. <!!!important!!!> 根据每则AI技术新闻...在标题开头添加一个不同的emoji表情包。
4. 必须包含AI技术新闻、AI学术论文、AI开源项目的链接。
```

`prompt`（用户消息，第 117-127 行）定数据加工规则：

```
- 在{{articles}}{{articles1}}{{articles2}}{{articles3}}中提取整理关于AI、大模型...的文章标题和对应链接为"AI技术新闻";
- 在{{arxiv}}中总结整理论文内容...整理为"AI学术论文"；
- 在{{GitHub}}中筛选最亮眼的5条AI开源项目...
# Attention
- 输出10条AI技术新闻\5条AI学术论文\5条AI开源项目
```

上游六路抓回来的是原始 JSON 列表，**过滤、去重、拣选、排版这些"脏活"没有写一行代码，全部externalize 给了提示词**。模型选的是 `DeepSeek-V3.2`（第 104 行），`maxTokens: 32768`（第 84 行）——日报很长，上限要给够。此外该节点还挂了一个 `get_current_datetime` 插件（第 62-67 行），供模型取"当天时间"写进日报抬头。

最后 `结束` 节点（第 34-48 行）把 `{{output}}` 绑定到 LLM 节点的 `output` 字段，`streamingOutput: true` 让日报打字机式流出——和 chapter4 里 `llm_client.py` 手写的流式循环，在这里是一个开关。

---

## 🎯 剥完之后：它在本章的位置

Coze 案例展示了低代码工作流的**最小完整闭环**：触发 → 并行取数 → LLM 加工 → 流式输出。它没有分支（对比 Dify 的问题分类器九路分发）、没有智能体自主决策（对比 n8n 的 AI Agent 自己决定调不调工具）——每一步都是**确定性的编排**，LLM 只在最后一站当"编辑"。

| 对比维度 | Coze 本例 | 其他三例 |
|---|---|---|
| 拓扑 | 扇出-扇入（无分支） | n8n 星型 / Dify 九分支 / FastGPT 四分支流水线 |
| LLM 的角色 | 末端汇总编辑 | n8n、Dify、FastGPT 里 LLM 还负责路由/决策 |
| 数据引用 | `ref_node` + `{{变量}}` | Dify 用 `{{#节点id.字段#}}`，FastGPT 用 `{{$节点id.字段$}}` |

先看懂这张菱形图，再去看后面三张更复杂的图，就只是"加分支、加循环、加自治"的事了。
