# 后端骨架 剥洋葱式讲解

> 源文件：`code/chapter15/Helloagents-AI-Town/backend/` 下的 `main.py`（393 行）、`config.py`（42 行）、`models.py`（69 行）、`logger.py`（115 行）、`view_logs.py`（113 行）
> 一句话概括：**用 FastAPI 搭起 AI 小镇的"市政厅"——对外提供 HTTP 接口，对内组装 NPC 管理器、状态管理器和日志系统，让 Godot 游戏客户端和 LLM 智能体各干各的。**

---

## 🧅 第一层（最外层）：这几个文件在干什么？

前面章节的智能体都活在命令行里：运行脚本、看输出、结束。本章要让智能体活在**游戏世界**里——玩家用键盘操纵角色，走到 NPC 面前按 E 聊天。问题来了：游戏引擎（Godot）不会跑 Python，LLM 也不该嵌进游戏进程。

答案是经典的**前后端分离**：Python 侧起一个 HTTP 服务，游戏侧只管发请求、收 JSON。这五个文件就是这个 HTTP 服务的骨架，各司其职：

| 文件 | 角色 |
|---|---|
| `main.py` | 前台接待：定义所有 API 路由，把请求转给对应的管理器 |
| `config.py` | 户籍科：集中管理端口、模型、更新间隔等配置 |
| `models.py` | 公文格式：用 Pydantic 定义请求/响应的数据结构 |
| `logger.py` | 档案室：把每次对话的全过程写进日志文件 |
| `view_logs.py` | 阅览室：一个独立小工具，实时查看日志（`tail -f` 的 Python 版） |

注意骨架里**没有一行 LLM 调用**——所有智能都藏在 `agents.py` 等模块里，骨架只负责"接线"。

---

## 🧅 第二层：`config.py`——一个类就是全部配置（第 6-41 行）

```python
class Settings:
    API_PORT = 8000
    NPC_UPDATE_INTERVAL = 30  # NPC状态更新间隔(秒)
    LLM_MODEL_ID: str = os.getenv("LLM_MODEL_ID", "Qwen/Qwen2.5-72B-Instruct")
    LLM_API_KEY: Optional[str] = os.getenv("LLM_API_KEY")
    LLM_BASE_URL: str = os.getenv("LLM_BASE_URL", "https://api-inference.modelscope.cn/v1/")
    CORS_ORIGINS = ["*"]
```

和 chapter4 的 `llm_client.py` 一脉相承：LLM 三件套（模型 ID、密钥、服务地址）从环境变量读取，换供应商不改代码。两个新面孔值得注意：

- **`NPC_UPDATE_INTERVAL = 30`**——NPC 自主对话每 30 秒批量刷新一次，这个数字直接决定了 API 成本（后面"关系与状态篇"会算这笔账）；
- **`CORS_ORIGINS = ["*"]`**——允许任意来源跨域。Godot 导出成 HTML5 后运行在浏览器里，没有这行，浏览器会直接拦掉所有请求。

`validate()`（第 27-39 行）在启动时检查密钥，但**缺密钥只警告不退出**——整个系统被设计成"没有 LLM 也能跑"（降级到预设对话），这是贯穿全章的容错哲学。

---

## 🧅 第三层：`models.py`——用 Pydantic 定协议（第 7-69 行）

```python
class ChatRequest(BaseModel):
    npc_name: str = Field(..., description="NPC名称")
    message: str = Field(..., description="玩家消息")
```

四个模型对应四种"公文"：`ChatRequest` / `ChatResponse`（一问一答）、`NPCInfo` / `NPCListResponse`（花名册）、`NPCStatusResponse`（NPC 自主对话的批量快照）。

Pydantic 模型在这里干了三件事：**校验**（Godot 发来的 JSON 缺字段直接 422，不会污染业务代码）、**文档**（FastAPI 靠它自动生成 `/docs` 交互页面，`json_schema_extra` 里的示例会原样出现在文档中）、**契约**（前端 GDScript 和后端 Python 隔着 HTTP 也要"说同一种话"，这份契约写在这里）。

`NPCStatusResponse`（第 46-63 行）是最有本章特色的一个：`dialogues` 是"NPC 名字 → 一句自言自语"的字典——它就是玩家在小镇里看到 NPC 头顶冒出的气泡文字的来源。

---

## 🧅 第四层：`main.py` 的 `lifespan`——服务的开机仪式（第 17-45 行）

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    settings.validate()
    npc_manager = get_npc_manager()
    state_manager = get_state_manager(settings.NPC_UPDATE_INTERVAL)
    await state_manager.start()
    yield
    await state_manager.stop()
```

`lifespan` 是 FastAPI 的生命周期钩子：`yield` 之前是开机，之后是关机。开机顺序有讲究——先验证配置，再初始化 NPC 管理器（创建三个 Agent 和记忆系统），最后启动状态管理器的**后台定时任务**（每 30 秒批量生成一次 NPC 自主对话）。

`get_npc_manager()` / `get_state_manager()` 都是**全局单例**工厂：三个 NPC 的记忆和好感度都存在管理器实例里，如果每个请求都新建一个，NPC 就会"每次见面都失忆"。单例保证了整个服务生命周期里"同一个张三"。

---

## 🧅 第五层：路由的分工——即时对话 vs 批量状态（第 103-179 行）

`main.py` 的十来个路由可以分成三组：

**即时对话组**——`POST /chat`（第 103-134 行）：玩家按 E 打字，走这条路。先查 NPC 存不存在（404），再把消息转给 `npc_mgr.chat()`，那里面才是记忆检索、LLM 生成、好感度分析的重头戏。

**批量状态组**——`GET /npcs/status`（第 149-163 行）和 `POST /npcs/status/refresh`（第 165-179 行）：Godot 每 30 秒轮询一次，拿到的是**缓存好的**批量对话，这个接口本身零 LLM 调用、毫秒级返回。

**观察调试组**——`GET /npcs/{name}/memories`、`GET /affinities`、`PUT /npcs/{name}/affinity` 等（第 199-377 行）：把 NPC 的记忆和好感度"开天窗"暴露出来，方便开发时观察 NPC 的内心世界，甚至直接改好感度做实验。

所有路由都是同一个套路：**验证 → 委托给管理器 → 包装成响应模型**，路由函数自己不含任何业务逻辑。

---

## 🧅 第六层：`logger.py` 与 `view_logs.py`——给对话过程装监控

`logger.py` 用标准库 `logging` 搭了一个专用通道（第 21-42 行）：`dialogue` logger 同时写文件（按日期命名 `dialogue_2026-07-02.log`）和控制台，`propagate = False` 防止日志被 root logger 重复打印。

它对外暴露的不是通用的 `log()`，而是**十个语义化函数**：`log_dialogue_start`、`log_affinity`、`log_memory_retrieval`、`log_affinity_change`……每个对应对话流程中的一个阶段。这样 `agents.py` 里的调用读起来就像流程注释，而一次完整对话在日志里是一段结构化的"病历"：

```
💬 对话开始: 张三 <-> 玩家
💖 当前好感度: 55.0/100 (友好)
🧠 检索到3条相关记忆
💬 张三回复: ...
📈 好感度变化: 55.0 -> 61.0 (+6.0)
```

`view_logs.py` 是配套的命令行工具：`python view_logs.py tail` 用"seek 到文件尾 + 循环 readline"（第 28-40 行）实现了跨平台的 `tail -f`，让你开一个终端跑服务、另一个终端实时看 NPC 的"内心独白"。

---

## 🧅 洋葱心：`/chat` 路由的三行委托（第 109-128 行）

```python
npc_info = npc_mgr.get_npc_info(request.npc_name)
if not npc_info:
    raise HTTPException(status_code=404, ...)

response_text = npc_mgr.chat(request.npc_name, request.message)

return ChatResponse(npc_name=request.npc_name, npc_title=npc_info["title"],
                    message=response_text, success=True)
```

整个后端骨架的精髓浓缩在这里：**路由层薄得几乎透明**。它只做三件事——验证、委托、包装。玩家消息进来时，路由完全不知道（也不需要知道）背后会发生记忆检索、LLM 推理、情感分析、好感度更新这一整条流水线；它只知道"把话递给张三，把张三的回话递回去"。

这种"薄路由、厚管理器"的分层，让 Godot 前端、FastAPI 骨架、Agent 内核三者可以独立演化——这正是综合项目区别于前面单文件示例的核心工程能力。

---

## 🎯 剥完之后：它在本章的位置

后端骨架是 AI 小镇的"承重墙"：

| 模块 | 它为谁服务 |
|---|---|
| `main.py` 路由 | Godot 的 `api_client.gd` 按这些 URL 发请求 |
| `models.py` 契约 | 前后端 JSON 格式的唯一权威定义 |
| `config.py` | `agents.py` 的 LLM、`state_manager.py` 的定时器都从这里取参数 |
| `logger.py` | `agents.py` 的六步对话流水线全程打点 |

骨架立好之后，剩下的问题才是本章真正的主角：张三、李四、王五如何"记得你、喜欢你、有自己的生活"——请看下一篇 `town_agents.md`。
