# AI 开发学习计划

> 适用于：前端工程师，零 AI / Python 基础，目标是上手公司 AI 项目开发
> 分析范围：a007-opensource、a007-holywell、thetagendata、holyeval

---

## 一、项目全景分析

### 1. a007-opensource（Mirobody）— 共享基础库

| 维度 | 详情 |
|------|------|
| **定位** | 开源健康数据平台核心库，被 a007-holywell 以 symlink 方式引用 |
| **技术栈** | Python 3.12 + FastAPI + Starlette + Uvicorn |
| **AI 技术** | LangChain, DeepAgents, MCP 协议 |
| **LLM 支持** | Gemini 3 Pro, GPT-5.1, Claude (via OpenRouter), DeepSeek, 火山引擎, DashScope |
| **数据库** | PostgreSQL 17 (pgvector 向量搜索) + Redis |
| **难度** | ★★★★☆ |

**核心模块：**

```
mirobody/
├── server/          # HTTP 服务核心（FastAPI 路由、JWT 中间件、限流）
├── chat/            # 对话管理（会话、历史、消息序列化、Agent 适配器）
│   ├── service.py   # ChatService 统一对话编排（33KB）
│   ├── adapters/    # LLM 适配器实现
│   ├── agents/      # Agent 定义
│   └── super_agent/ # 高级 Agent 编排
├── pub/
│   ├── agents/      # AI Agent 实现
│   │   ├── deep_agent.py      # DeepAgent（高级，支持任务规划、文件操作、MCP 工具）
│   │   ├── baseline_agent.py  # BaselineAgent（轻量，Gemini 原生 MCP）
│   │   └── synergy_agent/     # 多 Agent 协作
│   ├── tools/       # MCP 工具（25+ 图表可视化、通用工具）
│   ├── tools_health/# 健康领域工具（指标追踪、基因数据分析）
│   └── skills/      # 20+ 可扩展技能（代码审查、数据库设计、LLM 应用开发等）
├── pulse/           # 健康数据管道
│   ├── core/        # 数据标准化（单位转换、指标定义、聚合规则）
│   ├── data_upload/ # 数据摄入服务
│   ├── file_parser/ # 文件解析引擎（PDF/CSV/Excel/图片/音频/基因数据）
│   │   ├── handlers/          # 各格式处理器（工厂模式）
│   │   └── services/          # LLM 智能提取、食物识别、基因处理
│   ├── apple/       # Apple Health 集成
│   └── theta/       # Theta 平台（Garmin/Whoop/PostgreSQL 数据源）
├── utils/
│   └── llm/         # LLM 统一接口层
│       ├── config.py          # 多 Provider 配置管理
│       ├── adapters/          # 适配器（Gemini/OpenAI/火山引擎/工厂模式）
│       └── interface.py       # 统一 LLM 接口
├── mcp/             # MCP 协议服务（工具发现、资源管理）
└── user/            # 用户管理（JWT/OAuth/分享）
```

**你需要关注的重点：**
- `pub/agents/` — 理解 Agent 是怎么工作的
- `utils/llm/` — 理解多模型适配的设计模式
- `mcp/` — 理解 MCP 工具注册和调用机制
- `pulse/file_parser/` — 理解 LLM 如何用于文件内容提取

---

### 2. a007-holywell — 核心业务平台

| 维度 | 详情 |
|------|------|
| **定位** | 健康数据处理与 AI 分析平台（公司核心产品后端） |
| **技术栈** | Python 3.11+ / FastAPI / Docker / PostgreSQL 17 / Redis |
| **AI 技术** | LangChain ≥0.4, LangGraph ≥0.4, DeepAgents, MCP, mem0ai |
| **LLM 支持** | OpenAI (GPT-4/5/o1/o3), Gemini, Claude, AWS Bedrock, 百川, 火山引擎 |
| **监控** | Prometheus + OpenTelemetry + Sentry |
| **测试** | 134 个集成测试（YAML 声明式测试框架） |
| **难度** | ★★★★★ |

**核心架构：**

```
backend_py/
├── holywell/                # 主应用
│   ├── main.py              # 入口（PostgreSQL/Redis/MCP 初始化）
│   ├── middleware.py         # 请求中间件（trace_id、认证、上下文）
│   └── app/
│       ├── agents/          # AI Agent 实现
│       │   ├── hollywell/   # 领域 Agent（episode、plan graph、chat_v4）
│       │   ├── deep/        # DeepAgent 实现
│       │   └── langchain_agent.py  # LLM 模型工厂 + Agent 封装
│       └── holywell/
│           ├── routers/     # 25+ API 路由模块
│           │   ├── chat_v2_router.py    # 核心：AI 健康咨询对话
│           │   ├── medicine_router.py   # 药物数据库 + 交互检查
│           │   ├── journal.py           # 健康日记（文字+语音）
│           │   ├── file_upload.py       # 医疗文档上传（71KB+）
│           │   ├── doctor_visit.py      # 就医管理
│           │   └── ...                  # 食物、提醒、积分、搜索等
│           ├── services/    # 44 个业务服务模块
│           │   ├── chat/               # 对话处理、过滤器、中间件
│           │   ├── journal/            # 日记 CRUD + LLM 洞察提取
│           │   ├── medicine_service.py  # 药物服务（168KB+）
│           │   └── ...
│           ├── entity/      # 数据访问层（BaseRepository 模式）
│           ├── schema/      # Pydantic 请求/响应模型
│           └── evaluator/   # QA 评估、合规检查
│
├── mcp_server/              # MCP Agent 集成层
│   ├── agents/              # Agent 实现
│   │   ├── dynamic_agent.py       # 动态工具加载 Agent
│   │   └── middleware/            # DynamicToolMiddleware（Token 感知）
│   ├── tools/               # 15+ 健康领域 MCP 工具
│   │   ├── indicator_service.py   # 健康指标查询（3 个版本）
│   │   ├── journal_service.py     # 日记查询
│   │   ├── fda_drug_service.py    # FDA 药物信息
│   │   ├── clinical_trials_service.py  # 临床试验
│   │   └── medical_literature_service.py  # 医学文献
│   └── resources/           # MCP 资源
│
├── backend_job/             # 后台任务调度（APScheduler）
├── data_server/             # 数据访问服务（临床论文 Agent）
├── utils/                   # 共享工具
│   ├── llm/                 # LLM 集成
│   ├── metrics/             # Prometheus 指标
│   ├── utils_db.py          # 数据库操作（900+ 行）
│   └── utils_websocket.py   # WebSocket 管理
└── config/                  # 配置管理

backend_db/                  # 数据库
├── resource_holywell/       # 60+ SQL 迁移文件
└── scripts/                 # 维护脚本

api_tests/                   # 集成测试
├── tests/yaml/              # 20+ YAML 测试文件（134 个用例）
└── framework/               # 自定义 YAML 测试框架
```

**关键请求流程：**

```
HTTP 请求 → middleware（trace_id、认证）→ router → service → entity → PostgreSQL/Redis
                                                      ↓
                                              需要 AI 时
                                                      ↓
                                        Agent Selector（选择专家类型）
                                                      ↓
                                        LangChain Agent + MCP Tools
                                                      ↓
                                        LLM（GPT/Gemini/Claude）
                                                      ↓
                                        Filter（输出校验）→ SSE/轮询 响应
```

**你需要关注的重点：**
- `app/agents/langchain_agent.py` — LLM 模型工厂，理解多模型切换
- `mcp_server/tools/` — 健康领域 MCP 工具，理解工具如何被 Agent 调用
- `mcp_server/agents/dynamic_agent.py` — 动态 Agent，理解工具中间件
- `app/holywell/routers/chat_v2_router.py` — 核心对话路由
- `api_tests/` — 通过测试用例理解 API 行为

---

### 3. thetagendata — 合成健康数据生成

| 维度 | 详情 |
|------|------|
| **定位** | LLM 驱动的合成健康数据生成平台 |
| **技术栈** | Python 3.8+ / SQLAlchemy / Pydantic v2 |
| **AI 技术** | Google Gemini API（主力）, OpenAI 兼容 API（备用） |
| **数据库** | PostgreSQL（theta_ai 数据库） |
| **难度** | ★★★☆☆ |

**项目结构：**

```
thetagendata/
├── synthetic_generator.py     # 主入口（CLI）
├── medical_processor.py       # 医疗数据处理入口
├── configs/                   # YAML 配置文件
│   ├── synthetic_full.yaml        # 主配置（语言、模型、生成参数）
│   ├── synthetic_full_zh.yaml     # 中文配置
│   ├── synthetic_full_en.yaml     # 英文配置
│   └── database_writer_config.yaml # 数据库写入配置
├── indicators/                # 健康指标数据源
│   ├── th_series_dim.csv          # 主指标定义（2MB, 17K+ 指标）
│   └── standard/                  # FHIR 标准化指标
├── prompts/                   # LLM Prompt 模板（多语言）
│   ├── zh/synthetic/              # 中文生成 Prompt（15+ 文件）
│   └── en/synthetic/              # 英文生成 Prompt
├── scripts/
│   ├── foundation/
│   │   ├── llm/                   # LLM 客户端管理
│   │   │   ├── manager.py         # 统一 LLM 接口（自动选择、故障转移）
│   │   │   ├── gemini.py          # Gemini 实现
│   │   │   └── openai_compatible.py  # OpenAI 兼容实现
│   │   ├── prompts.py             # Prompt 模板管理
│   │   └── pydantic_models.py     # 数据模型
│   ├── shared_components/
│   │   ├── data_generator.py      # LLM 驱动的数据生成
│   │   ├── database_writer.py     # 数据库持久化
│   │   └── randomness.py          # 随机性控制（可复现）
│   └── synthetic_pipeline/
│       ├── enhanced_pipeline.py   # 核心统一管道
│       ├── incremental_pipeline.py # 增量生成（v2.0，支持断点续跑）
│       └── components/
│           ├── profile_generator.py      # 健康档案合成（3 阶段）
│           ├── profile_analyzer.py       # 档案语义分析
│           ├── health_data_generator.py  # 设备/体检数据生成
│           └── dynamic_events_synthesizer.py  # 事件驱动合成（v2.0）
├── tools/                     # 独立工具
│   ├── question_generator.py      # 个性化问题生成
│   ├── database_writer.py         # 数据库写入脚本
│   └── convert_medical_annotation.py  # 医生标注文件生成（Word/Excel）
└── outputs/                   # 生成数据输出目录
```

**数据生成管道：**

```
Stage 1: 档案生成
  LLM → 人口统计（年龄、性别、职业）→ 性格特征 → 健康档案（疾病、用药、家族史）

Stage 2: 指标选择
  健康档案 + 指标库(17K+) → LLM 语义匹配 → 选出设备指标(10-20个) + 体检指标(10-20个)

Stage 3: 设备数据生成
  LLM → 血压/心率/睡眠等连续监测数据（考虑日间波动、个体差异）

Stage 4: 体检数据生成
  LLM → 血液检查/影像报告等离散数据（考虑与设备数据的一致性）

Stage 5: 数据校验与导出
  → JSON（档案）+ CSV（指标数据）+ Word/Excel（医生标注）

Stage 6 (可选): 动态事件合成（v2.0）
  LLM → 饮食变化/运动计划/健康事件 → 事件驱动的指标变化

Stage 7 (可选): 问题生成
  LLM → 基于档案生成 5 个个性化健康问题

Stage 8 (可选): 写入数据库
  → PostgreSQL theta_ai 表
```

**你需要关注的重点：**
- `scripts/foundation/llm/manager.py` — 最简洁的 LLM 客户端封装，适合入门
- `prompts/` — 学习如何编写高质量 Prompt
- `configs/synthetic_full.yaml` — 理解 YAML 配置驱动的设计
- `scripts/synthetic_pipeline/components/` — 理解组件化管道设计

---

### 4. holyeval — AI 医疗助手评估框架

| 维度 | 详情 |
|------|------|
| **定位** | 虚拟用户评估框架，用于测试 AI 医疗助手的质量 |
| **技术栈** | Python 3.11+ / uv 工作区 / FastAPI / Next.js 16 |
| **AI 技术** | LangChain, OpenAI, Gemini, OpenRouter (300+ 模型) |
| **数据集** | 8 个基准数据集，9,517 个测试用例 |
| **前端** | hma-web (Next.js 16 + React 19 + Tailwind v4 + Prisma) |
| **难度** | ★★★★☆ |

**Monorepo 结构（uv workspace）：**

```
holyeval/
├── evaluator/               # 核心评估引擎
│   ├── core/
│   │   ├── orchestrator.py      # 核心！do_single_test() + BatchSession
│   │   ├── schema.py            # 25+ Pydantic 模型（Discriminated Union）
│   │   └── interfaces/          # 3 个 Agent 抽象基类（插件注册机制）
│   ├── plugin/
│   │   ├── test_agent/          # 虚拟用户 Agent
│   │   │   ├── auto.py          # LLM 驱动（模拟自然用户行为）
│   │   │   └── manual.py        # 脚本驱动（预定义输入序列）
│   │   ├── target_agent/        # 被测系统 Agent
│   │   │   ├── llm_api.py       # 通用 LLM API
│   │   │   └── theta_api.py     # Theta Health API
│   │   └── eval_agent/          # 11 个评估器！
│   │       ├── semantic.py          # 语义评估（LLM-as-Judge）
│   │       ├── healthbench.py       # 基于评分标准的医疗评估
│   │       ├── medcalc.py           # 医学计算准确性
│   │       ├── hallucination.py     # 幻觉检测（含 NCBI/CrossRef 验证）
│   │       ├── redteam_compliance.py # 红队合规测试
│   │       ├── memoryarena.py       # Agent 多任务记忆评估
│   │       └── ...                  # keyword, indicator, preset_answer 等
│   ├── utils/
│   │   ├── llm.py               # do_execute() — 统一 LLM 调用接口
│   │   ├── benchmark_reader.py  # 基准数据加载
│   │   └── checkpoint.py        # 断点续跑支持
│   └── tests/                   # 测试套件
│
├── benchmark/               # 基准数据与运行器
│   ├── data/                # 8 个数据集
│   │   ├── healthbench/         # 5,000 例（医疗 AI 综合评估）
│   │   ├── medcalc/             # 1,100 例（医学计算）
│   │   ├── agentclinic/         # 122 例（多科室临床诊断）
│   │   ├── medhall/             # 30 例（医疗幻觉检测）
│   │   ├── aq_redteam/          # 50 例（红队对抗测试）
│   │   ├── memoryarena/         # 701 例（Agent 记忆能力）
│   │   ├── extraction/          # 9 例（健康数据提取）
│   │   └── thetagen/            # 2,500 例（自定义评估）
│   ├── report/              # 评估报告输出
│   └── basic_runner.py      # CLI 运行器（断点续跑 + 并发控制）
│
├── generator/               # 数据转换工具（6 个数据集生成器）
│
├── web/                     # 内部开发 UI（FastAPI + Jinja2 + htmx）
│   └── app/
│       └── services/
│           └── task_manager.py  # BatchSession 封装 + SSE 实时推送
│
└── hma-web/                 # 公开排行榜（独立项目）
    ├── 技术栈: Next.js 16 + React 19 + TypeScript + Tailwind v4
    ├── 数据库: Prisma ORM + PostgreSQL
    ├── 认证: NextAuth.js v5
    ├── 动画: Framer Motion
    └── 页面: 排行榜 / 数据集 / 方法论 / 提交 / 管理后台
```

**三 Agent 评估循环：**

```
┌──────────────────────────────────────────────────┐
│  TestAgent（虚拟用户）                              │
│  - LLM 驱动 或 脚本驱动                           │
│  - 模拟患者提问行为                               │
└──────────────────┬───────────────────────────────┘
                   │ 多轮对话
                   ▼
┌──────────────────────────────────────────────────┐
│  TargetAgent（被测系统）                            │
│  - 通用 LLM API 或 Theta Health API              │
│  - 维护多轮对话状态                               │
└──────────────────┬───────────────────────────────┘
                   │ 对话结束
                   ▼
┌──────────────────────────────────────────────────┐
│  EvalAgent（评估器，11 种）                         │
│  - 语义 / 医学评分 / 幻觉检测 / 红队 / 记忆 ...   │
│  → 输出: score(0-1) + pass/fail + 详细反馈        │
└──────────────────────────────────────────────────┘
```

**插件注册机制（`__init_subclass__`）：**

```python
# 继承即注册，零配置
class AutoTestAgent(AbstractTestAgent, name="auto"):
    async def _generate_next_reaction(self, target_reaction):
        ...

# 通过名称查找
agent_cls = AbstractTestAgent.get("auto")
```

**你需要关注的重点：**
- `evaluator/core/schema.py` — Pydantic Discriminated Union 数据建模
- `evaluator/core/orchestrator.py` — 核心评估编排逻辑
- `evaluator/utils/llm.py` — 最清晰的统一 LLM 调用封装
- `hma-web/` — **Next.js 16 项目，你可以直接上手！**

---

## 二、项目关系图

```
┌─────────────────────────────────────────────────────────────┐
│                     a007-opensource (Mirobody)                │
│              共享基础库：Agent、LLM、MCP、健康数据管道         │
└─────────────┬───────────────────────────────────────────────┘
              │ symlink 引用
              ▼
┌─────────────────────────────────────────────────────────────┐
│                     a007-holywell                            │
│         核心业务平台：对话、药物、日记、文件、就医             │
│         ┌─────────┐  ┌─────────┐  ┌──────────┐             │
│         │ MCP 工具 │  │ Agent 层│  │ 业务路由  │             │
│         └─────────┘  └─────────┘  └──────────┘             │
└──────────────┬──────────────────────────┬───────────────────┘
               │                          │
    生成测试数据 │                          │ 评估 AI 质量
               ▼                          ▼
┌──────────────────────┐    ┌─────────────────────────────────┐
│    thetagendata       │    │          holyeval                │
│  合成健康数据生成      │    │   AI 医疗助手评估框架            │
│  → 输出到 theta_ai DB │    │   9,517 测试用例 / 11 评估器     │
│                       │    │   ┌─────────────────────┐       │
│                       │    │   │ hma-web (Next.js 16) │       │
│                       │    │   │ 公开排行榜 ← 你的切入点│       │
│                       │    │   └─────────────────────┘       │
└──────────────────────┘    └─────────────────────────────────┘
```

---

## 三、核心技术栈速查

| 技术 | JS/TS 对应物 | 在哪些项目中使用 | 优先级 |
|------|-------------|----------------|--------|
| **Python 基础** | JavaScript | 全部 4 个项目 | 必学 |
| **Pydantic v2** | Zod / io-ts | 全部 4 个项目 | 必学 |
| **FastAPI** | Express / Nest.js | a007-opensource, a007-holywell, holyeval | 必学 |
| **async/await (Python)** | async/await (JS) | 全部 4 个项目 | 必学 |
| **uv / pip** | npm / pnpm | 全部 4 个项目 | 必学 |
| **LangChain** | LangChain.js | a007-opensource, a007-holywell, holyeval | 核心 |
| **LLM API (OpenAI/Gemini)** | 同 | 全部 4 个项目 | 核心 |
| **MCP 协议** | 同 (有 JS SDK) | a007-opensource, a007-holywell | 核心 |
| **LangGraph** | — | a007-holywell | 进阶 |
| **PostgreSQL + pgvector** | — | a007-holywell, thetagendata | 进阶 |
| **SQLAlchemy** | Prisma / TypeORM | a007-holywell, thetagendata | 进阶 |
| **Next.js 16** | 你已经会了！ | holyeval (hma-web) | 直接上手 |
| **Prisma** | 你已经会了！ | holyeval (hma-web) | 直接上手 |

---

## 四、4 周学习计划

### 第 1 周：Python 基础 + LLM API 入门

**目标：** 能读懂项目 Python 代码，能独立调用 LLM API

| 天 | 内容 | 实操 |
|----|------|------|
| Day 1 | Python 基础语法（对照 JS） | 类型提示 ≈ TypeScript，装饰器 ≈ HOC，列表推导 ≈ map/filter |
| Day 2 | Python async/await + 异步编程 | 和 JS 几乎一样，重点学 `asyncio.gather()` ≈ `Promise.all()` |
| Day 3 | `uv` 包管理 + 虚拟环境 | `uv init` / `uv add` / `uv run` 跑通一个 hello world |
| Day 4 | Pydantic v2 数据模型 | 对照 Zod 学：BaseModel、Field、validator、Discriminated Union |
| Day 5 | OpenAI Chat Completions API | 用 Python 写一个最简聊天脚本 |
| Day 6 | Google Gemini API | 参考 `thetagendata/scripts/foundation/llm/gemini.py` |
| Day 7 | 阅读 thetagendata 的 LLM 封装 | `scripts/foundation/llm/manager.py` — 最简洁的多模型管理 |

> **核心对照表：**
> | Python | JavaScript/TypeScript |
> |--------|----------------------|
> | `pip` / `uv` | `npm` / `pnpm` |
> | `venv` | `node_modules` |
> | `requirements.txt` / `pyproject.toml` | `package.json` |
> | `def foo(x: int) -> str` | `function foo(x: number): string` |
> | `@decorator` | HOC / 装饰器提案 |
> | `[x for x in list if x > 0]` | `list.filter(x => x > 0)` |
> | `async def` / `await` | `async function` / `await` |
> | `asyncio.gather()` | `Promise.all()` |
> | `Pydantic BaseModel` | `Zod schema` |

---

### 第 2 周：FastAPI + Prompt 工程

**目标：** 能写简单的 API 服务，理解 Prompt 设计

| 天 | 内容 | 实操 |
|----|------|------|
| Day 1 | FastAPI 快速入门 | 对照 Express：路由、中间件、依赖注入、请求验证 |
| Day 2 | FastAPI 异步 + WebSocket | 理解 a007-holywell 的 SSE/WebSocket 对话流 |
| Day 3 | 深入阅读 thetagendata 的 Prompt 模板 | `prompts/zh/synthetic/` — 学习如何写高质量 Prompt |
| Day 4 | Pydantic Structured Output | LLM 返回 JSON → Pydantic 验证 → 结构化数据 |
| Day 5 | 阅读 thetagendata 的生成管道 | `scripts/synthetic_pipeline/components/` — 组件化设计 |
| Day 6 | 尝试跑通 thetagendata | 配置 `.env` + `python synthetic_generator.py --count 1` |
| Day 7 | 理解 YAML 配置驱动的设计 | `configs/synthetic_full.yaml` — 配置如何控制生成行为 |

---

### 第 3 周：LangChain + MCP + Agent

**目标：** 掌握 Agent 框架核心概念，理解 MCP 工具调用

| 天 | 内容 | 实操 |
|----|------|------|
| Day 1 | LangChain 核心概念 | ChatModel, Prompt Template, Output Parser, Chain |
| Day 2 | LangChain Tool Calling | 函数调用 ≈ MCP 工具调用，理解 Agent 如何选择工具 |
| Day 3 | MCP 协议入门 | 阅读 `a007-opensource/mirobody/mcp/` — 工具注册 + 资源管理 |
| Day 4 | 阅读 a007-holywell 的 MCP 工具 | `mcp_server/tools/` — 健康指标、药物、文献等工具实现 |
| Day 5 | 阅读 a007-opensource 的 Agent | `pub/agents/baseline_agent.py`（轻量版，先看这个） |
| Day 6 | 阅读 a007-holywell 的 Agent | `mcp_server/agents/dynamic_agent.py` — 动态工具加载 |
| Day 7 | 阅读 holyeval 的 LLM 封装 | `evaluator/utils/llm.py` — `do_execute()` 统一接口 |

---

### 第 4 周：评估框架 + 实际贡献

**目标：** 理解评估体系，开始贡献代码

| 天 | 内容 | 实操 |
|----|------|------|
| Day 1 | 阅读 holyeval 的数据模型 | `evaluator/core/schema.py` — Discriminated Union 模式 |
| Day 2 | 阅读 holyeval 的编排引擎 | `evaluator/core/orchestrator.py` — 三 Agent 评估循环 |
| Day 3 | 阅读 holyeval 的插件机制 | `__init_subclass__` 自注册 — 对比 JS 的插件模式 |
| Day 4 | 尝试跑通 holyeval 评估 | `python -m benchmark.basic_runner healthbench sample` |
| Day 5-6 | **上手 hma-web 开发** | Next.js 16 + React 19 + Tailwind v4 — 你的舒适区！ |
| Day 7 | 选择一个实际任务开始贡献 | 见下方「切入建议」 |

---

## 五、切入建议（按推荐优先级排序）

### 立即可以做的（前端技能直接适用）

1. **holyeval/hma-web** — Next.js 16 + React 19 + Tailwind v4 + Prisma
   - 公开排行榜平台，技术栈完全在你的舒适区
   - 页面：排行榜、数据集浏览、提交表单、管理后台
   - 可以立即开始 UI 开发、性能优化、新功能开发

2. **holyeval/web** — FastAPI + htmx + Alpine.js
   - 内部开发 UI，轻量前端
   - 适合学习 FastAPI 模板渲染 + SSE 实时推送

### 短期目标（1-2 周后可以做）

3. **thetagendata** — 最适合 AI 入门
   - 代码结构清晰，组件化设计
   - LLM 封装最简洁（`manager.py` 只有 400 行）
   - Prompt 模板丰富，适合学习 Prompt 工程
   - 修改配置 → 调整 Prompt → 观察输出，迭代快

4. **a007-holywell 的 MCP 工具** — 写新工具
   - MCP 工具是独立的 Python 函数，代码量小
   - 每个工具只做一件事，容易理解和修改

### 中期目标（3-4 周后可以做）

5. **holyeval 评估器插件** — 写新评估器
   - 插件机制清晰（继承 + 自注册）
   - 每个评估器独立，不影响其他代码

6. **a007-holywell 的 Agent 层** — 参与 Agent 开发
   - 需要理解 LangChain + MCP 全流程
   - 建议先从读代码 → 修 bug → 加小功能开始

---

## 六、推荐资源

| 类别           | 资源                               | 说明                                  |
| ------------ | -------------------------------- | ----------------------------------- |
| Python 入门    | Python for JavaScript Developers | 对比式学习，最快上手                          |
| Pydantic     | docs.pydantic.dev                | 你会 Zod 就很快，重点看 Discriminated Union  |
| FastAPI      | fastapi.tiangolo.com             | 交互式文档，2 小时入门                        |
| LangChain    | python.langchain.com/docs        | 先看 quickstart + tool calling        |
| LangChain JS | js.langchain.com                 | 如果想先用 TS 理解概念                       |
| MCP 协议       | modelcontextprotocol.io          | 官方规范 + SDK 文档                       |
| OpenAI API   | platform.openai.com/docs         | Chat Completions + Function Calling |
| Gemini API   | ai.google.dev                    | Google AI Studio 可在线测试              |
| Prompt 工程    | 直接看 `thetagendata/prompts/`      | 项目内的 Prompt 就是最好的学习材料               |
