# Chapter 13 代码剥洋葱讲解

对 `code/chapter13/helloagents-trip-planner/` 综合项目的逐层拆解。每篇文档都采用"剥洋葱"结构：从最外层的**这个模块在干什么**开始，一层层剥到最核心的**那几行关键代码**，最后回到全局，说明它在全栈智能体应用中的位置。

## 推荐阅读顺序

按一次请求流经系统的顺序读：先看骨架，再看灵魂，然后是给灵魂供血的服务层与国境线般的 API 层，最后到用户眼前的前端：

| 顺序 | 文档 | 源文件 | 主题 | 一句话 |
|---|---|---|---|---|
| 1 | [后端入口与配置.md](后端入口与配置.md) | `backend/run.py`、`app/config.py`、`app/api/main.py` | 后端骨架 | 点火、供电、装配：uvicorn 启动 + Pydantic Settings + CORS 与路由注册 |
| 2 | [智能体核心.md](智能体核心.md) | `app/agents/trip_planner_agent.py` | 多智能体流水线（本章灵魂） | 四个提示词四个专家：搜景点→查天气→找酒店→规划，解析瀑布 + Pydantic 把 LLM 输出锁成 JSON |
| 3 | [服务层.md](服务层.md) | `app/services/`（llm / amap / unsplash） | 外部能力封装 | 三个外部世界三个单例：LLM 客户端、高德 MCP 子进程、Unsplash REST |
| 4 | [API路由与数据模型.md](API路由与数据模型.md) | `app/api/routes/` + `app/models/schemas.py` | 接口门面与数据合同 | 路由薄如纸、模型是安检机：`TripRequest` 进关、`TripPlan` 出关、温度校验器洗数据 |
| 5 | [前端.md](前端.md) | `frontend/src/`（App / Home / Result / api / types） | Vue3 前端 | 表单收需求、假进度条讲真剧情、LLM 的经纬度落成高德地图上的图钉 |

## 一条主线

本章讲的是**全栈智能体应用——AI 旅行规划师**。前面章节的智能体在终端里自娱自乐，本章把它装进一个真实产品：用户点一次按钮，走完这条链路——

> **前端表单**（Home.vue 收集城市/日期/偏好）
> → **axios POST `/api/trip/plan`**（120 秒超时，专为 LLM 流水线而设）
> → **FastAPI 路由**（TripRequest 安检进关）
> → **四个 Agent 顺序接力**（景点/天气/酒店三个采集专家调高德 MCP 拿真实数据，规划专家纯推理编排）
> → **解析瀑布 + Pydantic 校验**（把 LLM 的自由文本锁成 TripPlan，失败则 fallback 兜底）
> → **前端渲染**（Result.vue：行程卡片、预算表、高德地图上的图钉与每日路线）

贯穿全部五篇的一条工程铁律：**对 LLM 既要给足约束，又要假设它随时违约**——提示词里贴完整 JSON 模板（软约束），代码里备好三级解析瀑布、温度清洗器、坐标判空和占位行程（硬约束）。同一份数据形状在提示词、Pydantic 模型、TypeScript 接口中出现三次并保持镜像，这条"形状链"就是全栈应用不散架的原因。

> 前置阅读：chapter4 的讲解（`code/chapter4/讲解/`）解释了 SimpleAgent 背后"提示词定协议，代码做解析"的思想；本章是这一思想在多智能体、真实产品尺度上的总演习。
