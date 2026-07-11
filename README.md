<h1 align="center">SQLBot 配煤专家问答系统</h1>

<p align="center">
  <a href="#-项目简介">项目简介</a> •
  <a href="#-核心特性">核心特性</a> •
  <a href="#-目录结构">目录结构</a> •
  <a href="#-快速部署">快速部署</a> •
  <a href="#-使用指南">使用指南</a> •
  <a href="#-技术架构">技术架构</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.11-blue?style=flat-square&logo=python" alt="Python">
  <img src="https://img.shields.io/badge/FastAPI-0.115-green?style=flat-square&logo=fastapi" alt="FastAPI">
  <img src="https://img.shields.io/badge/Vue-3.5-brightgreen?style=flat-square&logo=vue.js" alt="Vue">
  <img src="https://img.shields.io/badge/PostgreSQL-16-blue?style=flat-square&logo=postgresql" alt="PostgreSQL">
  <img src="https://img.shields.io/badge/License-GPL--3.0-orange?style=flat-square" alt="License">
</p>

---

## 📖 项目简介

基于 **SQLBot** 开源框架构建的配煤行业专家问答系统，集成配煤领域专业知识库，通过自然语言交互实现对话式数据分析（ChatBI）。

本项目针对配煤行业场景深度定制，内置配煤专家 RAG 问答语料库，支持配煤比例优化、煤质分析、成本核算等专业领域的智能问答与数据可视化。

### 🎯 适用场景

- 配煤方案智能推荐与优化
- 煤质数据自然语言查询
- 配煤成本分析与预测
- 煤炭库存管理问答
- 配煤工艺知识检索

---

## ✨ 核心特性

<table>
<tr>
<td width="50%">

### 🤖 智能问答
- 自然语言转 SQL
- 配煤领域专业术语识别
- RAG 增强问答准确度
- 多轮对话上下文保持

</td>
<td width="50%">

### 📊 数据可视化
- 自动生成可视化图表
- 支持多种图表类型
- 交互式数据探索
- 仪表盘自定义配置

</td>
</tr>
<tr>
<td width="50%">

### 🔐 企业级能力
- 工作空间隔离
- 行级数据权限
- 操作审计日志
- 多数据源接入

</td>
<td width="50%">

### 🚀 灵活部署
- Docker 一键部署
- 多种集成方式
- MCP 协议支持
- 嵌入式应用接入

</td>
</tr>
</table>

---

## 📁 目录结构

```
SQLBot/
├── 📂 src/                              # SQLBot 核心源码
│   ├── 📂 backend/                      # FastAPI 后端服务
│   │   ├── 📂 apps/                     # 业务模块
│   │   │   ├── chat/                    # 对话问答
│   │   │   ├── datasource/              # 数据源管理
│   │   │   ├── terminology/             # 术语库（RAG）
│   │   │   ├── data_training/           # 训练样例
│   │   │   ├── dashboard/               # 仪表盘
│   │   │   └── system/                  # 系统管理
│   │   ├── 📂 common/                   # 公共模块
│   │   ├── 📂 alembic/                  # 数据库迁移
│   │   ├── main.py                      # 应用入口
│   │   └── pyproject.toml               # Python 依赖配置
│   ├── 📂 frontend/                     # Vue 3 前端应用
│   │   ├── 📂 src/
│   │   │   ├── views/                   # 页面组件
│   │   │   ├── components/              # 通用组件
│   │   │   ├── router/                  # 路由配置
│   │   │   └── stores/                  # 状态管理
│   │   └── package.json                 # 前端依赖配置
│   ├── 📂 g2-ssr/                       # 图表渲染服务（Node.js）
│   ├── docker-compose.yaml              # 容器编排配置
│   └── README.md                        # 上游项目文档
├── 📂 data/                             # 运行时数据（不纳入版本控制）
│   ├── postgresql/                      # PostgreSQL 数据文件
│   └── sqlbot/                          # SQLBot 运行数据
│       ├── excel/                       # Excel 数据源
│       ├── file/                        # 文件存储
│       ├── images/                      # 图表图片
│       └── logs/                        # 应用日志
├── 📂 others/                           # 扩展资源
│   └── 配煤专家知识库_RAG问答语料.xlsx   # 配煤领域 RAG 语料库
├── 📄 SQLBot-项目详解.md                # 技术架构深度解析文档
└── 📄 README.md                         # 本文档
```

### 文件说明

| 文件/目录 | 说明 |
|-----------|------|
| `src/` | SQLBot 开源框架源码，包含完整的前后端与图表渲染服务 |
| `data/` | 运行时生成的数据文件，包括数据库数据、日志、图片等（**不应纳入 Git**）|
| `others/配煤专家知识库_RAG问答语料.xlsx` | 配煤行业专业术语、问答对、SQL 训练样例等 RAG 语料 |
| `SQLBot-项目详解.md` | SQLBot 架构逐层拆解文档，包含启动流程、问答链路、RAG 机制等技术细节 |

---

## 🚀 快速部署

### 环境要求

- **操作系统**: Linux / macOS / Windows
- **Docker**: >= 20.10
- **内存**: >= 4GB
- **磁盘**: >= 10GB

### 方式一：Docker 一键部署（推荐）

```bash
# 1. 克隆本仓库
git clone https://github.com/FeynmanNddbb/coal-blend-chatbi.git
cd coal-blend-chatbi

# 2. 获取 SQLBot 源码（如果 src/ 目录不存在）
git clone https://github.com/dataease/SQLBot.git src

# 3. 启动服务
cd src
docker run -d \
  --name sqlbot \
  --restart unless-stopped \
  -p 8000:8000 \
  -p 8001:8001 \
  -v $(pwd)/../data/sqlbot/excel:/opt/sqlbot/data/excel \
  -v $(pwd)/../data/sqlbot/file:/opt/sqlbot/data/file \
  -v $(pwd)/../data/sqlbot/images:/opt/sqlbot/images \
  -v $(pwd)/../data/sqlbot/logs:/opt/sqlbot/app/logs \
  -v $(pwd)/../data/postgresql:/var/lib/postgresql/data \
  --privileged=true \
  dataease/sqlbot

# 4. 检查服务状态
docker logs -f sqlbot
```

### 方式二：Docker Compose 部署

```bash
cd src
docker-compose up -d
```

### 方式三：源码部署

#### 后端服务

```bash
cd src/backend

# 安装依赖（需要 Python 3.11）
pip install uv
uv sync

# 配置环境变量（可选）
cp .env.example .env
# 编辑 .env 配置数据库连接、Redis 等

# 启动服务
uv run python main.py
```

#### 前端服务

```bash
cd src/frontend

# 安装依赖
npm install
# 或使用 pnpm: pnpm install

# 开发模式
npm run dev

# 生产构建
npm run build
```

#### 图表渲染服务

```bash
cd src/g2-ssr
npm install
npm start
```

### 访问应用

| 服务 | 地址 | 说明 |
|------|------|------|
| **Web 控制台** | http://localhost:8000 | 主应用界面 |
| **MCP Server** | http://localhost:8001 | Model Context Protocol 服务 |

**默认登录信息:**
- 用户名: `admin`
- 密码: `SQLBot@123456`

> ⚠️ **首次登录后请立即修改默认密码**

---

## 📚 使用指南

### 1. 配置大模型

登录后依次进入：**系统管理** → **AI 模型配置** → **添加模型**

支持的模型服务商：

| 服务商 | API 兼容性 | 推荐模型 |
|--------|-----------|---------|
| OpenAI | 原生 | GPT-4, GPT-3.5-turbo |
| 阿里云百炼 | OpenAI 兼容 | Qwen-Max, Qwen-Plus |
| DeepSeek | OpenAI 兼容 | deepseek-chat, deepseek-coder |
| 腾讯混元 | OpenAI 兼容 | hunyuan-pro, hunyuan-lite |
| 讯飞星火 | OpenAI 兼容 | spark-v3.5, spark-v3.1 |
| Kimi | OpenAI 兼容 | moonshot-v1 |

配置项：
- **模型类型**: 选择对应服务商
- **API Key**: 从服务商获取的密钥
- **API Domain**: API 基础地址（如使用代理或自部署服务）
- **模型名称**: 具体模型标识符

### 2. 添加数据源

**系统管理** → **数据源管理** → **添加数据源**

支持的数据库类型：
- MySQL / MariaDB
- PostgreSQL
- SQL Server
- Oracle
- ClickHouse
- Hive
- Elasticsearch
- Excel（自动转换为 PostgreSQL）

配置完成后，系统会自动：
- 抓取表结构和字段元数据
- 生成表和字段的 embedding 向量
- 建立 RAG 召回索引

### 3. 导入配煤专家知识库

**工作空间设置** → **术语库管理** → **导入**

选择 `others/配煤专家知识库_RAG问答语料.xlsx` 导入，语料库包含：

- **术语库**: 配煤行业专业术语及解释（如"挥发分"、"灰熔点"、"配煤比"等）
- **训练样例**: 典型问题与对应 SQL 的示例对
- **自定义提示词**: 针对配煤场景的 Prompt 优化

导入后系统会自动触发 embedding 计算，可在日志中查看进度。

### 4. 开始问答

进入**对话问答**界面，可以这样提问：

```
- 查询本月各煤种的采购量和平均价格
- 哪些配煤方案的灰分在 10%-15% 之间？
- 对比上季度和本季度的配煤成本趋势
- 计算当前库存中挥发分大于 30 的煤种总量
- 推荐热值在 5000-5500 大卡的最优配煤方案
```

系统会自动：
1. 从术语库召回相关专业术语
2. 从训练样例中找到类似问题的 SQL 模板
3. 选择相关数据表和字段
4. 生成 SQL 并执行
5. 自动出图（柱状图、折线图、饼图等）
6. 给出分析结论和预测

### 5. 创建仪表盘

**仪表盘管理** → **新建仪表盘** → **画布编辑**

- 拖拽添加多个问答结果
- 自由布局和尺寸调整
- 设置自动刷新
- 分享或嵌入到其他系统

---

## 🏗️ 技术架构

### 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                     前端层 (Vue 3 + Vite)                    │
│  对话界面 · 数据源管理 · 仪表盘 · 系统配置 · 嵌入式应用      │
└────────────────────────┬────────────────────────────────────┘
                         │ REST API / SSE
┌────────────────────────▼────────────────────────────────────┐
│                   应用层 (FastAPI)                           │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  LLM 编排引擎 (14步流程)                             │    │
│  │  选数据源 → 选表 → RAG召回 → 生成SQL → 执行 → 出图    │    │
│  └─────────────────────────────────────────────────────┘    │
│  术语库 · 训练样例 · 数据源管理 · 权限控制 · 审计日志        │
└────────┬──────────────────────────────────┬────────────────┘
         │                                  │
┌────────▼──────────┐            ┌──────────▼─────────────────┐
│  PostgreSQL +     │            │   业务数据库                │
│  pgvector         │            │   MySQL / Oracle / ...      │
│  (元数据·向量库)   │            │   (查询分析的实际数据)       │
└───────────────────┘            └─────────────────────────────┘
         │
    ┌────▼────┐  ┌──────────────┐  ┌─────────────┐
    │  Redis  │  │  g2-ssr      │  │ MCP Server  │
    │ (缓存)  │  │ (图表渲染)    │  │  :8001      │
    └─────────┘  └──────────────┘  └─────────────┘
```

### 核心技术栈

#### 后端
- **框架**: FastAPI 0.115
- **语言**: Python 3.11
- **ORM**: SQLAlchemy 2.0
- **LLM 编排**: LangChain
- **向量检索**: pgvector
- **缓存**: Redis (fastapi-cache2)
- **依赖管理**: uv

#### 前端
- **框架**: Vue 3.5 + TypeScript
- **构建工具**: Vite 6
- **UI 组件**: Element Plus
- **图表库**: AntV G2 / S2 / X6
- **状态管理**: Pinia
- **路由**: Vue Router (hash 模式)

#### 数据库
- **元数据库**: PostgreSQL 16 + pgvector
- **支持接入**: MySQL / Oracle / SQL Server / ClickHouse / Hive / Elasticsearch

#### 其他
- **图表渲染**: Node.js + g2-ssr
- **容器化**: Docker + Docker Compose
- **MCP 协议**: 支持 Claude Desktop / Cursor 集成

---

## 📄 深入了解

想深入理解 SQLBot 的技术实现？阅读完整的架构解析文档：

👉 **[SQLBot-项目详解.md](./SQLBot-项目详解.md)**

文档包含：
- 启动流程与中间件分层
- 14 步问答完整链路
- LLM / Embedding 抽象设计
- RAG 混合检索机制
- 行级权限与动态数据源
- MCP 集成方案
- 关键设计亮点与工程经验

---

## 🤝 贡献指南

欢迎贡献配煤领域的专业知识和改进建议！

### 如何贡献

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 提交 Pull Request

### 贡献内容

- 🔧 配煤领域术语补充
- 📝 SQL 训练样例优化
- 🐛 Bug 修复
- 📚 文档改进
- 💡 新功能建议

---

## 📝 License

- **本仓库内容**（配煤语料、技术文档）: MIT License
- **SQLBot 源码** (`src/` 目录): [GPL-3.0 + Fit2Cloud 附加条款](src/LICENSE)

---

## 🔗 相关链接

- [SQLBot 官方仓库](https://github.com/dataease/SQLBot)
- [DataEase 开源 BI](https://github.com/dataease/dataease/)
- [Model Context Protocol](https://modelcontextprotocol.io/)

---

<p align="center">
  <b>如果这个项目对你有帮助，请给个 ⭐ Star 支持一下！</b>
</p>

<p align="center">
  有问题或建议？欢迎提 <a href="../../issues">Issue</a> 或 <a href="../../pulls">Pull Request</a>
</p>
