# Chapter 5 代码剥洋葱讲解

对 `code/chapter5/` 下四个低代码平台工作流导出文件的逐层拆解。本章没有 Python 代码——四个文件分别是 Coze、n8n、Dify、FastGPT 平台上"画"出来的智能体应用的导出配置（YAML/JSON）。每篇文档仍采用"剥洋葱"结构：从最外层的**这张图在干什么**开始，一层层剥到最核心的**那几个字段**，最后横向对比它在四个平台版图中的位置。

## 推荐阅读顺序

按工作流拓扑复杂度递增排列——从"一条直线"剥到"多专家流水线"：

| 顺序 | 文档 | 源文件 | 主题 | 一句话 |
|---|---|---|---|---|
| 1 | [HelloAgent_cozeCase.md](HelloAgent_cozeCase.md) | `HelloAgent_cozeCase.zip` | Coze AI 日报生成器 | 最小闭环：六路信源扇出并行抓取，扇入一个 LLM 编成日报 |
| 2 | [HelloAgent_n8nCase.md](HelloAgent_n8nCase.md) | `HelloAgent_n8nCase.json` | n8n 邮件自动回复助手 | 单智能体+能力插槽：Gmail 轮询触发，Agent 插上模型/记忆/搜索/RAG 四个"插头" |
| 3 | [HelloAgent_difyCase.md](HelloAgent_difyCase.md) | `HelloAgent_difyCase.yml` | Dify 超级智能个人助手 | 意图路由：问题分类器九路分发，含 MCP 智能体、text2SQL 链和异步轮询循环 |
| 4 | [HelloAgent_fastgptCase.md](HelloAgent_fastgptCase.md) | `HelloAgent_fastgptCase.json` | FastGPT 智能投顾小助手 | 多专家流水线：表单画像 → 风险/技术面/基本面/舆情四位分析师接力 → 可下载报告 |

> 关于 zip：`HelloAgent_cozeCase.zip` 解压后是可读的 YAML（`MANIFEST.yml` + `workflow/AI_news-draft.yaml`），因此同样单独成篇，讲解基于解压后的内容。

## 一条主线

本章讲的是**不写代码，如何组织智能体**。四个平台界面千差万别，导出文件剥开后却是同一副骨架：

> **节点 = 函数**（每个节点声明输入、输出、配置）
> **边 = 数据流**（Coze 的 `ref_node`、Dify 的 `{{#id.字段#}}`、FastGPT 的 `{{$id.字段$}}`、n8n 的 `$('节点')` ——四种语法，一个思想：显式声明"我的输入来自谁"）
> **提示词 = 藏在节点字段里的"代码"**（Coze 在 `llmParam.systemPrompt`，n8n 在 `options.systemMessage`，Dify 在 `prompt_template` / `agent_parameters.instruction`，FastGPT 在 `inputs[].systemPrompt`——找到这个字段，就找到了应用的灵魂）

四个案例还各自贡献了一块拼图，合起来正是 chapter4 手写范式的"图形化转世"：

| 案例 | 独有机制 | 对应的手写概念 |
|---|---|---|
| Coze | 扇出-扇入并行聚合 | 并发调用 + 汇总提示词 |
| n8n | `main` 边与 `ai_*` 边分离的能力插槽 | 依赖注入（LLM/记忆/工具作为构造参数） |
| Dify | 问题分类器 + 循环/条件分支/等待 | 意图识别 + `while` 轮询异步任务 |
| FastGPT | 专家串行接力 + 表单 + 多模态交付 | Plan-and-Solve 的静态化（计划画死在图上） |

贯穿四个文件的仍是那条工程铁律：**提示词定协议，下游做解析**——n8n 要求 Agent 输出严格 JSON 供发信节点消费，Dify 用参数提取器从文本抠 `task_id`，FastGPT 让每位分析师按固定格式交报告给下一位。变的只是"下游"从解析代码换成了下一个节点。

> 前置阅读：chapter4 的讲解（`code/chapter4/讲解/`）手写了 ReAct / Plan-and-Solve / Reflection 三种范式；本章把同样的思想搬上画布——看懂了手写版，这些 YAML/JSON 里的每个字段都会觉得眼熟。
