# SQLBot 项目详解

> 版本：v1.8.0  
> 协议：GPL-3.0 + Fit2Cloud 附加条款  
> 出品方：DataEase / Fit2Cloud  
> 文档定位：基于源码与 README 的逐层拆解，帮助你快速建立"为什么这么写"的工程直觉。

---

## 目录

1. 项目定位与核心价值
2. 整体架构
3. 后端启动流程（lifespan）
4. 一次问答的完整调用链路
5. AI 模型与 Embedding 抽象
6. RAG 召回机制（术语 / 训练 / 数据源 / 表）
7. 数据库执行层
8. 行级权限与动态数据源
9. MCP（Model Context Protocol）集成
10. 安全与中间件
11. 前端结构
12. 核心数据模型
13. 部署与运行
14. 推荐的源码阅读路径
15. 关键设计亮点速记

---

## 1. 项目定位与核心价值

SQLBot 是一款 **ChatBI（对话式商业智能）** 系统：用户用自然语言提问，系统自动生成 SQL → 执行 → 出图 → 给出分析与预测，同时支持仪表盘、嵌入式集成、MCP Server 等多种使用形态。

核心特性：

- **自然语言转 SQL**：基于 LangChain + 多步 Prompt 编排，将用户问题转化为可执行的 SQL，并把结果渲染为图表。
- **多 LLM 兼容**：通过 OpenAI 协议抽象，支持阿里云百炼、千帆、DeepSeek、腾讯混元、讯飞星火、Gemini、Kimi、火山引擎、MiniMax、vLLM、Azure OpenAI 等。
- **多数据库支持**：MySQL / PostgreSQL / SQL Server / Oracle / ClickHouse / Redshift / 达梦 / Hive / Elasticsearch / Excel→PG。
- **RAG 增强**：术语库、训练样例、数据源、表 schema 全部 embedding 化，问答时按相似度召回，避免一次性塞满 prompt。
- **企业能力**：工作空间隔离、行级权限、审计日志、动态 CORS、商业 xpack 包（监控、加密等）。
- **多端集成**：Web 控制台、iframe 嵌入、悬浮助手、MCP Server（标准协议供 Claude / Cursor 等使用）。

---

## 2. 整体架构

```
┌──────────────────── Frontend (Vue 3 + Vite) ────────────────────┐
│  /login  /chat  /dashboard  /canvas  /system/*  /set/*           │
│  /assistant  /embeddedPage  /chatPreview  ...                    │
│  Pinia · Vue Router(hash) · Element Plus · AntV G2/S2/X6         │
└──────────────────────────────┬───────────────────────────────────┘
                               │ REST / SSE
┌──────────────────────────────▼───────────────────────────────────┐
│                      FastAPI 主应用 (port 8000)                   │
│  middleware: CORS → Token → Response → RequestContext × 2         │
│  router:  /api/v1/{login,user,workspace,chat,datasource,...}      │
│                                                                   │
│  ┌────────── apps/chat/task/llm.py  LLMService ──────────┐        │
│  │  init_messages → run_task                              │        │
│  │   ├─ select_datasource (RAG)                           │        │
│  │   ├─ choose_table_schema                               │        │
│  │   ├─ filter_terminology / filter_training              │        │
│  │   ├─ generate_sql / generate_with_sub_sql              │        │
│  │   ├─ check_sql → execute_sql                           │        │
│  │   ├─ generate_chart → request_picture (g2-ssr)         │        │
│  │   └─ analysis / predict / recommend                    │        │
│  └─────────────────────────────────────────────────────────┘      │
│                                                                   │
│  apps/ai_model         · LLMFactory + EmbeddingModelCache         │
│  apps/terminology      · 术语 RAG（pgvector + ILIKE）              │
│  apps/data_training    · SQL 训练样例 RAG                          │
│  apps/datasource       · 多数据库元数据 + embedding                 │
│  apps/system           · 用户、工作空间、模型、嵌入应用、审计        │
│  apps/dashboard        · 仪表盘组件 / 画布                          │
│  common/core           · 配置 / 缓存 / 响应封装                    │
└──────────────────────────────┬───────────────────────────────────┘
                               │
        ┌──────────────────────┴─────────────────────┐
        │                                            │
┌───────▼────────┐                          ┌────────▼─────────┐
│ PostgreSQL +   │  元数据 / 向量库          │  各业务数据库      │
│ pgvector       │                          │  MySQL / Oracle…  │
└────────────────┘                          └───────────────────┘
        │
┌───────▼────────┐    ┌────────────────────┐   ┌────────────────┐
│  Redis (cache) │    │ g2-ssr (Node 服务)  │   │ MCP Server :8001│
└────────────────┘    └────────────────────┘   └────────────────┘
```

模块按 **业务域** 划分（`apps/<domain>/{api,crud,models,schemas,...}`），公共能力沉淀在 `common/`。这种结构让"加一个新业务"和"换一个 LLM 协议"都只动一处。

---

## 3. 后端启动流程（`main.py` 的 `lifespan`）

`main.py:50-63` 用 `@asynccontextmanager` 定义启动钩子，顺序非常关键：

| 步骤 | 调用 | 作用 |
| --- | --- | --- |
| 1 | `run_migrations()` | Alembic 把 schema 升到 `head` |
| 2 | `init_sqlbot_cache()` | 初始化 fastapi-cache2（默认走 Redis）|
| 3 | `init_dynamic_cors(app)` | 从数据库读取已注册的嵌入应用域名，动态加入 CORS |
| 4 | `fill_empty_terminology_embeddings()` | 给历史术语补 embedding |
| 5 | `fill_empty_data_training_embeddings()` | 给历史训练样例补 embedding |
| 6 | `fill_empty_table_and_ds_embeddings()` | 给历史数据源 / 表补 embedding |
| 7 | `clean_xpack_cache()` | 商业包缓存清理 |
| 8 | `async_model_info()` | 把存量模型的明文 key/url 加密回写 |
| 9 | `monitor_app(app)` | 商业版监控接入 |

中间件按以下顺序叠加（注意 FastAPI 的中间件是 **后注册先执行**）：

```python
CORSMiddleware
TokenMiddleware            # 解析 JWT → request.state.user
ResponseMiddleware         # 统一返回结构 {code,data,msg}
RequestContextMiddleware   # 业务侧上下文（工作空间 oid 等）
RequestContextMiddlewareCommon  # 审计上下文
```

路由全部挂在 `settings.API_V1_STR` 之下；Swagger 通过 `custom_openapi` 实现 i18n（中英文 OpenAPI schema 各一份并缓存）。

---

## 4. 一次问答的完整调用链路

入口：`POST /api/v1/chat/question`（`apps/chat/api/chat.py`）。

### 4.1 入口处理

1. `parse_quick_command()` 识别 `/regenerate`、`/analysis`、`/predict`，转换为对应 `QuickCommand` + `OperationEnum`。
2. `find_base_question()` 沿 `regenerate_record_id` 链路递归回溯，确保多次重生成仍能拿到原始问题。
3. 历史消息只保留最近若干轮（`get_last_conversation_rounds`）。
4. 实例化 `LLMService`（async classmethod `create`）。

### 4.2 `LLMService.run_task` 编排（`apps/chat/task/llm.py`）

`OperationEnum` 共 14 步，主流程会按需调用其中若干步：

| 阶段 | 关键步骤 | 说明 |
| --- | --- | --- |
| 选数据源 | `CHOOSE_DATASOURCE` | 仅当用户没有显式选数据源时启用 |
| 选表 | `CHOOSE_TABLE` | 让 LLM 从候选表里挑出相关表 |
| RAG | `FILTER_TERMS` / `FILTER_SQL_EXAMPLE` / `FILTER_CUSTOM_PROMPT` | 用相似度召回术语、训练样例、自定义提示词 |
| 出 SQL | `GENERATE_SQL` / `GENERATE_SQL_WITH_PERMISSIONS` / `GENERATE_DYNAMIC_SQL` | 普通 SQL 或带行权限 / 动态过滤的 SQL |
| 校验 | `EXECUTE_SQL`（含 `check_sql`）| 真正执行前再让 LLM 自检语法、字段名 |
| 出图 | `GENERATE_CHART` / `GENERATE_PICTURE` | 先得到图表 DSL，再调用 g2-ssr 渲染成 PNG |
| 分析 / 预测 | `ANALYSIS` / `PREDICT_DATA` | 拿到查询结果后做文字解读或时序预测 |
| 推荐 | `GENERATE_RECOMMENDED_QUESTIONS` | 根据数据源给出"接下来你可以问" |

每一步都会写入 `ChatLog`，前端通过 SSE 接收 `step` 流式事件，UI 上呈现"思考中 → 选表 → 写 SQL → 执行 → 出图"。

### 4.3 流式输出与 reasoning

`process_stream` 同时处理两类思维链：

- 文本里夹的 `<think>...</think>` 标签（部分国产模型用法）
- `additional_kwargs.reasoning_content`（OpenAI o1 / DeepSeek-R1 类用法）

它会把 reasoning 与正文分离，分别推送到前端，避免污染最终答案。

### 4.4 图片渲染

`request_picture` 把图表 DSL POST 给 `MCP_IMAGE_HOST`（独立的 Node 服务 `g2-ssr`），返回 PNG 路径，再通过 FastAPI 静态目录 `mcp_app.mount("/images", ...)` 暴露。

---

## 5. AI 模型与 Embedding 抽象

### 5.1 `LLMConfig` + `LLMFactory`（`apps/ai_model/model_factory.py`）

```python
@dataclass(frozen=True)
class LLMConfig:
    model_type: str   # "openai" | "vllm" | "azure"
    model_name: str
    api_key: str
    api_domain: str | None
    ...
    def __hash__(self): ...
```

- **frozen + 自定义 `__hash__`**：让 `LLMConfig` 能作为 `lru_cache` 的 key。
- `LLMFactory.create_llm` 用 `@lru_cache(maxsize=32)` 缓存已创建的 `BaseLLM` 实例，避免反复初始化 SDK。
- `BaseLLM` 是抽象基类，子类有 `OpenAILLM` / `OpenAIvLLM` / `OpenAIAzureLLM`。所有国产模型只要兼容 OpenAI Chat Completions API，就走 `OpenAILLM` 这条路。
- `get_default_config(custom_model_id)` 从 `AiModelDetail` 表读取，解密 `api_key`、`api_domain` 后注入。

### 5.2 Embedding 单例（`apps/ai_model/embedding.py`）

```python
class EmbeddingModelCache:
    _instance = None
    _lock = threading.Lock()
```

- 双重检查锁的单例。
- 默认模型 `shibing624_text2vec-base-chinese`，本地加载，`device='cpu'`，`normalize_embeddings=True`（向量归一化，方便用 `<=>` 余弦距离）。
- 启动时把历史数据补 embedding，保证后续召回稳定可用。

---

## 6. RAG 召回机制

### 6.1 术语库（`apps/terminology/curd/terminology.py`）

混合检索：

1. **词法层**：`ILIKE %word%` 拿到字面命中（解决 embedding 对短词、专名召回弱的问题）。
2. **向量层**：pgvector `<=>` 余弦距离 + 阈值 `EMBEDDING_TERMINOLOGY_SIMILARITY` + `TOP_COUNT`。
3. 父子结构（`pid`）：召回子术语时把父术语一并带上，保持语义完整。
4. 命中结果通过 `to_xml_string` 序列化为 XML，注入 system prompt：

```xml
<terminology>
  <item><word>客单价</word><explain>...</explain></item>
</terminology>
```

### 6.2 训练样例（`apps/data_training`）

存放"问题 → SQL"对，思路与术语相同：embedding + 阈值召回，作为 few-shot 注入。

### 6.3 数据源选择（`apps/datasource/embedding/ds_embedding.py`）

把每个 `CoreDatasource.embedding` 与问题向量做余弦相似度，取 top `DS_EMBEDDING_COUNT`，让 LLM 在候选里挑。

### 6.4 表选择

类似地对 `CoreTable.embedding` 排序，再让 LLM 二次确认；这是"两阶段召回 + 重排"的典型做法。

---

## 7. 数据库执行层

支持的方言通过 `apps/datasource/` 下的 driver 适配实现，关键点：

- **元数据抓取**：`CoreField` 存字段类型、注释，用于 prompt 提示。
- **SQL 执行**：使用 SQLAlchemy / 原生 driver，按数据源类型分发；每条 SQL 在执行前会先经过 `check_sql`（让 LLM 自检语法）。
- **结果回流**：DataFrame → JSON / Excel 导出（`xlsxwriter`、`openpyxl`、`python-calamine`）。
- **Excel 数据源**：上传 → 解析 → 写入 PostgreSQL，再当作普通数据源使用。

---

## 8. 行级权限与动态数据源

- `OperationEnum.GENERATE_SQL_WITH_PERMISSIONS`：根据当前用户的角色拼出 `WHERE` 过滤条件。
- 动态数据源：`dynamic_ds_types = [1, 3]`，配合 `dynamic_subsql_prefix = 'select * from sqlbot_dynamic_temp_table_'`。
  - 思路：先把外部传入的"动态结果集"落到一个临时表（命名规则 `sqlbot_dynamic_temp_table_<id>`），再让 LLM 把它当成普通表去写 SQL。
  - 这样可以让助手 / 嵌入式应用把"上一步的查询结果"作为下一轮问答的输入，形成多轮分析。
- `generate_assistant_dynamic_sql`、`build_table_filter`、`generate_filter` 共同支撑这个流程。

---

## 9. MCP 集成

`main.py` 中：

```python
mcp = FastApiMCP(
    app,
    name="SQLBot MCP Server",
    include_operations=[
        "mcp_datasource_list",
        "get_model_list",
        "mcp_question",
        "mcp_start",
        "mcp_assistant",
        "mcp_ws_list",
    ],
)
mcp.mount(mcp_app)   # 挂到独立 FastAPI 实例
```

- **白名单**：只把这 6 个操作暴露给 MCP，避免内网管理接口外泄。
- **独立 app**：`mcp_app` 单独跑（注释里 `uvicorn.run("main:mcp_app", host="0.0.0.0", port=8001)`），可以独立鉴权和限流。
- 这样 Claude Desktop、Cursor 等任何 MCP 客户端都能把 SQLBot 当成一个"数据问答工具"接入。

---

## 10. 安全与中间件

| 层 | 实现 | 关键点 |
| --- | --- | --- |
| 认证 | `TokenMiddleware` | 解析 JWT，写入 `request.state.user` |
| 鉴权 | `RequestContextMiddleware` | 工作空间 `oid` 注入到每个查询 |
| 审计 | `RequestContextMiddlewareCommon` | 记录请求来源、用户、操作类型 |
| 加密 | `pycryptodome` + `cryptography` | 模型 api_key、api_domain 落库时加密 |
| 跨域 | `CORSMiddleware` + `init_dynamic_cors` | 静态白名单 + 嵌入应用动态注册 |
| XSS | 前端 `vue-dompurify-html` | 渲染 LLM 输出时净化 HTML |

特别注意：日志和异常处理在 `common/core/response_middleware.py`，统一返回 `{code, data, msg}`，并把内部异常脱敏后透出。

---

## 11. 前端结构

技术栈：Vue 3.5 + TypeScript + Vite 6 + Element Plus + Pinia + Vue Router（hash 模式）+ AntV G2/S2/X6 + TinyMCE + markdown-it + html2canvas + vue-dompurify-html。

### 11.1 入口

`src/frontend/src/main.ts`：

```ts
const app = createApp(App)
app.use(createPinia())
app.use(router)
app.use(i18n)
app.use(VueDOMPurifyHTML)
app.mount('#app')
```

极简，所有能力靠插件挂载。

### 11.2 路由（`src/frontend/src/router/index.ts`）

| 路径 | 用途 |
| --- | --- |
| `/login`、`/admin-login` | 登录 |
| `/chat/index` | 主问答界面 |
| `/dsTable/:dsId/:dsName` | 数据源表浏览 |
| `/dashboard/index` | 仪表盘列表 |
| `/canvas` | 仪表盘编辑器 |
| `/dashboard-preview` | 单仪表盘预览 |
| `/set/{member,permission,professional,training,prompt}` | 工作空间配置（成员/权限/术语/训练/提示词）|
| `/system/{user,workspace,model,embedded,setting/*,audit}` | 系统级管理 |
| `/assistant`、`/embeddedPage`、`/embeddedCommon` | 嵌入式入口（iframe / 悬浮）|
| `/chatPreview`、`/assistantTest` | 预览与调试 |
| `/401` | 无权限页 |

布局组件三种：

- `Layout`：常规带左侧菜单的全功能布局
- `LayoutDsl`：DSL 化的精简布局，主问答和工作空间在用
- `SinglePage`：单页（如表浏览器）

### 11.3 组件特点

- 图表统一通过 AntV G2/S2 渲染，dashboard 编辑器用 X6 做画布。
- TinyMCE 用于富文本说明（如自定义提示词）。
- `vue-dompurify-html` 包住所有 LLM 输出的 HTML，防 XSS。

---

## 12. 核心数据模型

### 12.1 对话域（`apps/chat/models/chat_model.py`）

- `Chat`：一次会话
- `ChatRecord`：一轮问答记录（含 `regenerate_record_id` 串成重生成链）
- `ChatLog`：每个 `OperationEnum` 步骤的日志，前端用它回放思考过程

枚举：

```python
class OperationEnum(IntEnum):
    GENERATE_SQL = 1
    GENERATE_CHART = 2
    ANALYSIS = 3
    PREDICT_DATA = 4
    GENERATE_RECOMMENDED_QUESTIONS = 5
    GENERATE_SQL_WITH_PERMISSIONS = 6
    CHOOSE_DATASOURCE = 7
    GENERATE_DYNAMIC_SQL = 8
    CHOOSE_TABLE = 9
    FILTER_TERMS = 10
    FILTER_SQL_EXAMPLE = 11
    FILTER_CUSTOM_PROMPT = 12
    EXECUTE_SQL = 13
    GENERATE_PICTURE = 14
```

`ChatFinishStep` 简化版，仅给前端展示进度条用：`GENERATE_SQL=1 / QUERY_DATA=2 / GENERATE_CHART=3`。

`AiModelQuestion` 是 Pydantic 模型，但同时承担"prompt 装配器"角色：

- `sql_sys_question` / `sql_user_question` / `chart_*` / `analysis_*` / `predict_*` / `datasource_*` / `guess_*` / `filter_*` / `dynamic_*` 一系列方法，把每个步骤的 system / user prompt 拼好。
- 自定义消息类 `SystemPromptMessage` / `HumanPromptMessage` / `AIPromptMessage` 加了 `sqlbot_system=True`，方便后续做"系统消息只发一次、用户消息流式"等区分。

### 12.2 数据源域（`apps/datasource/models/datasource.py`）

- `CoreDatasource`：`(id, name, type, configuration JSONB, oid, table_relation JSONB, embedding)`，`oid` 即工作空间 ID，是隔离的关键。
- `CoreTable`：`(ds_id, table_name, embedding)`
- `CoreField`：字段元数据
- `DsRecommendedProblem`：每个数据源的推荐问题

### 12.3 模型域

- `AiModelDetail`：保存模型协议、加密后的 key/url、参数等。

---

## 13. 部署与运行

### 13.1 后端

`pyproject.toml` 锁定 Python 3.11，使用 `uv` 管理依赖（含 `pytorch-cpu` / `pytorch-cu128` 双 index、`testpypi` 安装商业 `sqlbot-xpack`）。

启动：

```bash
cd src/backend
uv sync
uv run python main.py        # 主应用 0.0.0.0:8000
# 如需 MCP：取消注释跑 main:mcp_app on 8001
```

依赖的外部服务：

- PostgreSQL（必须装 pgvector 扩展）
- Redis（fastapi-cache2 用）
- g2-ssr（独立 Node 服务，专门做图表 PNG 渲染）

### 13.2 前端

```bash
cd src/frontend
npm install        # 或 pnpm
npm run dev        # vite dev
npm run build      # vue-tsc -b && vite build
npm run lint       # eslint --fix
```

注意 `dev` 也会跑 `vue-tsc -b`，类型错误会直接卡住启动。

### 13.3 一键部署

`src/README.md` 给出了 docker-compose 的一键启动方式，是最快的尝试路径。

---

## 14. 推荐的源码阅读路径

如果你只有几个小时，建议按这个顺序看：

1. **`src/backend/main.py`** — 看清启动顺序、中间件、MCP 挂载。
2. **`apps/api.py`** — 一眼扫过所有业务路由，建立模块地图。
3. **`apps/chat/api/chat.py`** — 找到 `/chat/question`，跟随入口看快捷指令解析、历史回溯。
4. **`apps/chat/task/llm.py`** — 主菜。重点看 `run_task` 的步骤编排、`process_stream`、`request_picture`。
5. **`apps/chat/models/chat_model.py`** — 看 `OperationEnum` 和 `AiModelQuestion` 的 prompt 装配。
6. **`apps/ai_model/model_factory.py` + `embedding.py`** — 理解 LLM / Embedding 的抽象层。
7. **`apps/terminology/curd/terminology.py`** — 看一个完整的 RAG 实现（混合检索 + XML 注入）。
8. **`apps/datasource/embedding/ds_embedding.py`** — 数据源 / 表的二级召回。
9. 前端从 `src/frontend/src/router/index.ts` → `/chat/index` 对应的视图开始往下看，配合后端 SSE 接口理解流式 UI。

---

## 15. 关键设计亮点速记

> 可以当成"工程经验卡"收藏。

- **Lifespan 严格顺序**：迁移 → 缓存 → CORS → 三类 embedding 补齐 → 商业初始化。任何顺序错乱都会引发"启动后第一次问答必失败"的诡异 bug。
- **frozen dataclass + `__hash__` 当 lru_cache key**：避免每次问答都重建 LangChain LLM 客户端，是 Python 里复用对象的标准花式做法。
- **EmbeddingModel 双重检查锁单例**：本地模型加载耗时长（百 MB 级），必须保证全进程一份。
- **多步 OperationEnum 编排**：把"端到端 NL2SQL"拆成 14 个原子步骤，每步都能单独打日志、单独失败重试，前端就能精细化展示进度。
- **混合检索（ILIKE + pgvector）**：纯向量在专名 / 短词上召回不稳；纯关键字又不懂语义。两条腿走路是当前业界默认方案。
- **MCP 白名单**：只暴露 6 个安全操作，是把"内部 API"和"对外可调用工具"明确切分的好例子。
- **动态数据源临时表**：用统一前缀 `sqlbot_dynamic_temp_table_` 把外部传入的中间结果"伪装成普通表"，从而让所有现有 SQL 生成逻辑无感复用。
- **reasoning 双通道兼容**：同时识别 `<think>` 与 `additional_kwargs.reasoning_content`，对接国产模型的常见私货格式。
- **Prompt 装配器塞进 Pydantic 模型**：`AiModelQuestion` 既做参数校验又做 prompt 拼装，单文件就是"问答协议 + 提示词模板"，方便统一维护。
- **i18n OpenAPI**：根据 `?lang=` 或 `Accept-Language` 给 Swagger 出中英文两份 schema 并缓存，对外协作时很省事。
- **加密回填**：启动时把存量明文 key 异步加密回写，是"安全性补丁"上线的优雅姿势。
- **前端只用 hash 路由**：避免 nginx 配置 `try_files` 的麻烦，部署到任何静态服务器都能跑，是 ToB 工具系统的稳妥选择。

---

> 阅读本文后，建议你做一遍"反向练习"：随便挑一个 `OperationEnum` 步骤，自己讲清楚它的 system prompt 是怎么拼的、走了哪条 RAG、最后落到了哪张表。能讲清楚一个步骤，整套链路就基本通了。
