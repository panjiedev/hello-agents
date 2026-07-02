# API 路由与数据模型 剥洋葱式讲解

> 源文件：`code/chapter13/helloagents-trip-planner/backend/app/api/routes/trip.py`（86 行）、`poi.py`（129 行）、`map.py`（163 行）、`backend/app/models/schemas.py`（206 行）
> 一句话概括：**三个路由文件定义了后端对外的全部"门面"，而 schemas.py 用 Pydantic 模型给每一扇门装上了"安检机"——进出的数据都必须过形状检查。**

---

## 🧅 第一层（最外层）：这四个文件在干什么？

前端和智能体之间需要一份"合同"：前端发什么格式的请求、后端还什么格式的响应。这份合同分两半写：

- **routes/**：写"有哪些门、每扇门干什么"——`/api/trip/plan` 生成行程、`/api/poi/photo` 取景点图、`/api/map/weather` 查天气……
- **schemas.py**：写"每扇门进出的数据长什么样"——`TripRequest`、`TripPlan`、`WeatherInfo`……

FastAPI 的精髓正是把两者焊在一起：路由函数的参数类型注解就是请求校验器，`response_model` 就是响应校验器。**合同不是文档，是可执行的代码。**

---

## 🧅 第二层：trip.py——主链路的入口（第 20-52 行）

```python
@router.post("/plan", response_model=TripPlanResponse, summary="生成旅行计划", ...)
async def plan_trip(request: TripRequest):
    ...
    agent = get_trip_planner_agent()      # 取多智能体系统单例
    trip_plan = agent.plan_trip(request)  # 四步流水线（见智能体核心篇）

    return TripPlanResponse(
        success=True,
        message="旅行计划生成成功",
        data=trip_plan
    )
```

刨掉打印日志，这个本章最重要的接口只有三步：**取单例 → 跑流水线 → 包响应**。路由层薄得像张纸——它不做任何业务，只做"翻译"：HTTP 请求翻译成 `TripRequest` 对象（FastAPI 自动完成），`TripPlan` 对象翻译回 JSON 响应。业务全在 Agent 层，这就是分层的纪律。

出错时（第 54-61 行）抛 `HTTPException(status_code=500, detail=...)`——FastAPI 会把它变成规范的错误响应，前端 `api.ts` 里的 `error.response?.data?.detail` 正好接住这个 `detail` 字段，**前后端的错误处理在这里对上了暗号**。

顺带一提，`/trip/health`（第 69-85 行）里的 `agent.agent.name` 引用了一个不存在的属性（`MultiAgentTripPlanner` 只有 `attraction_agent` 等四个成员）——真调用会落进 503 分支。读开源项目时能认出这种"改了架构没改健康检查"的痕迹，也是一种收获。

---

## 🧅 第三层：poi.py 与 map.py——两组辅助门面

**poi.py** 里最有意思的是给前端供图的 `/poi/photo`（第 94-128 行）：

```python
photo_url = unsplash_service.get_photo_url(f"{name} China landmark")

if not photo_url:
    # 如果没找到,尝试只用景点名称搜索
    photo_url = unsplash_service.get_photo_url(name)
```

一次小小的**检索降级**：先用 `"故宫 China landmark"` 这种加了限定词的查询提高命中质量，搜不到再裸搜景点名。前端 Result 页每张景点卡片的配图，都是从这扇门取的。

**map.py** 则是服务层的"直通车"：`/map/poi`、`/map/weather`、`/map/route` 三个接口几乎是 `AmapService` 对应方法的一比一转发（如第 105-132 行的 `plan_route`）。它的存在演示了一件事：**同一份地图能力，Agent 可以用（TOOL_CALL 路径），前端也可以直接用（HTTP 路径）**。三个文件的每个接口都是同一个骨架：try → 调服务 → 包成 `{success, message, data}` → except 抛 HTTPException，统一得像模具压出来的。

---

## 🧅 第四层：schemas.py——合同的具体条文（第 10-150 行）

请求侧的 `TripRequest`（第 10-33 行）展示了 Pydantic 的声明式校验：

```python
class TripRequest(BaseModel):
    city: str = Field(..., description="目的地城市", example="北京")
    travel_days: int = Field(..., description="旅行天数", ge=1, le=30, example=3)
    preferences: List[str] = Field(default=[], description="旅行偏好标签", ...)
```

`...` 表示必填，`ge=1, le=30` 直接把"天数 1 到 30"写进类型——前端如果传 50 天，请求根本进不了路由函数，FastAPI 自动回 422。`description` 和 `example` 还会出现在 `/docs` 的交互文档里，**一份定义，三处受益（校验、文档、示例）**。

响应侧是一棵嵌套的模型树，和规划 Agent 提示词里的 JSON 模板严格镜像：

```
TripPlan
 ├── days: List[DayPlan]
 │        ├── hotel: Optional[Hotel]
 │        ├── attractions: List[Attraction]  (含 location: Location)
 │        └── meals: List[Meal]
 ├── weather_info: List[WeatherInfo]
 └── budget: Optional[Budget]
```

注意可选性的设计：`hotel`、`budget` 是 `Optional`，`weather_info` 默认空列表——**LLM 没给的锦上添花字段不至于让整个计划校验失败**，而 `city`、`days`、`overall_suggestions` 这些骨架字段是必填的，缺了就该走 fallback。宽严有度。

---

## 🧅 洋葱心：温度校验器（schemas.py 第 119-130 行）

```python
@field_validator('day_temp', 'night_temp', mode='before')
@classmethod
def parse_temperature(cls, v):
    """解析温度,移除°C等单位"""
    if isinstance(v, str):
        v = v.replace('°C', '').replace('℃', '').replace('°', '').strip()
        try:
            return int(v)
        except ValueError:
            return 0
    return v
```

这十行是全文件最"有故事"的代码。规划 Agent 的提示词里明明白白写着"温度必须是纯数字(不要带°C等单位)"，但工程师依然在模型这一侧装了清洗器——因为 LLM 总有不听话的时候，`"25°C"`、`"25℃"` 都见过。

`mode='before'` 表示在类型转换**之前**执行：先把字符串洗成纯数字，再交给 `Union[int, str]` 的常规校验。配合字段类型声明里那个宽容的 `Union[int, str]`（第 114-115 行），形成一套完整的"接纳 → 清洗 → 规整"流水线。

它和智能体核心篇的解析瀑布共同构成本章最重要的工程主题的两道防线：

> **提示词侧**：告诉 LLM 该输出什么形状（软约束）
> **模型侧**：不管 LLM 输出什么，进门先过安检、能洗则洗、洗不了给默认值（硬约束）

软约束提高良品率，硬约束保证下限——两者缺一不可。

---

## 🎯 剥完之后：它在本章的位置

在"生成行程"的完整链路里，这一层是**前后端的国境线**：

> 前端 axios POST → **`TripRequest` 安检进关** → 路由转交 Agent 流水线 → LLM 输出 → 解析瀑布 → **`TripPlan` 安检出关**（温度清洗器在此上岗）→ JSON → 前端渲染

`schemas.py` 更是一鱼三吃：

| 消费方 | 用它做什么 |
|---|---|
| FastAPI 路由 | 请求/响应自动校验 + `/docs` 文档生成 |
| 规划 Agent 提示词 | JSON 模板与之镜像（约束 LLM 输出） |
| 前端 `types/index.ts` | TypeScript 接口与之一一对应（见前端篇） |

一份数据形状，从提示词到 Python 到 TypeScript 三处保持一致——**这条"形状链"就是全栈智能体应用不散架的原因**。
