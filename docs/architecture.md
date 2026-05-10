# MiroFish 架构与原理

## 概述

MiroFish 是基于多 Agent 的群体智能预测引擎。用户上传种子材料（新闻报告、政策文件、小说文本等），系统自动构建高保真的数字平行世界，让具有独立人格的 AI Agent 在其中自由交互演化，最终输出预测报告。

## 技术栈

| 层 | 技术 | 说明 |
|---|---|---|
| 前端 | Vue 3 + Vite + D3.js | SPA，D3 负责知识图谱可视化 |
| 后端 | Flask + Flask-CORS | REST API，异步任务管理 |
| LLM | OpenAI SDK 格式 | 兼容任意 OpenAI 格式 API（通义千问、DeepSeek 等） |
| 知识图谱 | Zep Cloud | GraphRAG，实体/关系存储与语义检索 |
| 社交模拟 | CAMEL-AI OASIS | Twitter / Reddit 双平台多 Agent 模拟 |
| 国际化 | vue-i18n + 后端 locale | 中英文双语 |

## 五步工作流

```
用户上传文件 + 描述需求
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│ Step 1: 图谱构建                                         │
│   上传文件 → 文本提取 → LLM生成本体 → Zep构建知识图谱      │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 2: 环境配置                                         │
│   图谱实体 → Agent人设生成 → LLM生成模拟参数              │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 3: 模拟运行                                         │
│   Twitter + Reddit 双平台并行 → Agent行为回写图谱         │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 4: 报告生成                                         │
│   ReACT Agent 自主检索 → 分章节生成 Markdown 报告         │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 5: 深度交互                                         │
│   与 Report Agent 对话 / 与模拟世界中任意 Agent 对话      │
└─────────────────────────────────────────────────────────┘
```

## 各步骤详解

### Step 1: 图谱构建

**入口**: `POST /api/graph/ontology/generate` → `POST /api/graph/build`

**流程**:

1. **文件上传与文本提取** (`FileParser`)
   - 支持 PDF、Markdown、TXT 格式
   - PDF 使用 PyMuPDF 提取，支持非 UTF-8 编码检测（charset-normalizer / chardet）

2. **本体生成** (`OntologyGenerator`)
   - 将文档文本发送给 LLM
   - LLM 分析文本，生成适合社交模拟的实体类型（人物、组织、事件等）和关系类型
   - 输出 JSON 格式的本体定义

3. **文本分块** (`TextProcessor`)
   - 按 chunk_size（默认 500 字符）分块，overlap 50 字符
   - 预处理：去除多余空白、规范化

4. **知识图谱构建** (`GraphBuilderService`)
   - 在 Zep Cloud 创建 Standalone Graph
   - 设置本体（entity_types + edge_types）
   - 分批（batch_size=3）添加文本块作为 episode
   - 等待 Zep 异步处理完成（轮询 episode 状态）
   - Zep 自动提取实体和关系，构建图谱

**关键点**: 图谱构建是异步任务，通过 `TaskManager` 管理，前端轮询 `GET /api/graph/task/<task_id>` 获取进度。

### Step 2: 环境配置

**入口**: `POST /api/simulation/configure`

**流程**:

1. **实体读取** (`ZepEntityReader`)
   - 从 Zep 图谱读取所有节点
   - 按预定义类型过滤实体（人物、组织等）
   - 获取每个实体的关联边和相关节点信息

2. **Agent 人设生成** (`OasisProfileGenerator`)
   - 将图谱实体转换为 OASIS Agent Profile
   - 每个 Agent 有独立的人格、背景、行为倾向
   - 支持 Twitter 和 Reddit 两种平台格式

3. **模拟参数生成** (`SimulationConfigGenerator`)
   - LLM 根据模拟需求和图谱数据，智能生成：
     - 时间线设计（模拟总轮数、关键时间节点）
     - 事件注入计划（什么时候注入什么事件）
     - Agent 行为配置（不同 Agent 的活跃度、倾向性）

### Step 3: 模拟运行

**入口**: `POST /api/simulation/start`

**流程**:

1. **模拟启动** (`SimulationRunner`)
   - 在独立子进程中运行 OASIS 模拟脚本
   - Twitter 和 Reddit 双平台并行模拟
   - 每个平台有不同的可用动作集：
     - Twitter: CREATE_POST, LIKE_POST, REPOST, FOLLOW, QUOTE_POST, DO_NOTHING
     - Reddit: CREATE_POST, CREATE_COMMENT, LIKE/DISLIKE, SEARCH, FOLLOW, MUTE 等

2. **进程间通信** (`SimulationIPC`)
   - Flask 主进程与模拟子进程通过文件系统 IPC 通信
   - 命令文件：主进程写入控制命令（暂停、恢复、停止）
   - 响应文件：子进程回传状态和行为记录
   - 每轮模拟结束后记录所有 Agent 的动作

3. **动态记忆更新** (`ZepGraphMemoryUpdater`)
   - 实时将 Agent 的行为转换为自然语言描述
   - 写入 Zep 图谱，形成演化的集体记忆
   - 后续轮次的 Agent 可以"感知"之前发生的事情

**关键点**: 模拟运行在子进程中，主进程通过 IPC 获取状态。前端轮询 `GET /api/simulation/<id>/status` 实时展示进度。

### Step 4: 报告生成

**入口**: `POST /api/report/generate`

**流程**:

1. **Report Agent** (`ReportAgent`)
   - 基于 ReACT（Reasoning + Acting）模式
   - Agent 自主规划报告大纲，决定需要检索哪些信息

2. **工具调用** (`ZepToolsService`)
   - **InsightForge**: 深度混合搜索，结合语义搜索和图谱遍历
   - **PanoramaSearch**: 广度搜索，扫描全局趋势和模式
   - **快速搜索**: 针对性查询特定实体或关系

3. **分章节生成**
   - Agent 按大纲逐章节生成 Markdown 内容
   - 每个章节完成后立即保存（`section_01.md`, `section_02.md`...）
   - 前端可轮询 `GET /api/report/<id>/sections` 实时展示已完成章节
   - 最终合并为完整报告

4. **日志记录**
   - 结构化 Agent 日志（每步推理、工具调用、结果）
   - 控制台风格日志（INFO/WARNING 级别）
   - 前端可实时查看 Agent 的思考过程

### Step 5: 深度交互

**入口**: `POST /api/report/chat`

- 用户与 Report Agent 对话，Agent 可自主调用检索工具回答问题
- 支持对话历史，多轮交互
- Agent 有完整的模拟世界上下文，可以回答关于模拟过程和结果的任何问题

## 核心服务模块

```
backend/app/services/
├── ontology_generator.py          # LLM 生成本体定义
├── graph_builder.py               # Zep 图谱构建
├── text_processor.py              # 文本提取、分块、预处理
├── zep_entity_reader.py           # 从图谱读取实体
├── oasis_profile_generator.py     # 生成 Agent 人设
├── simulation_config_generator.py # LLM 生成模拟参数
├── simulation_manager.py          # 模拟生命周期管理
├── simulation_runner.py           # 子进程运行模拟
├── simulation_ipc.py              # 文件系统 IPC 通信
├── zep_graph_memory_updater.py    # 模拟行为回写图谱
├── report_agent.py                # ReACT 报告生成 Agent
├── zep_tools.py                   # Agent 检索工具集
└── __init__.py
```

## API 路由

三个 Flask Blueprint：

### /api/graph — 图谱管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/ontology/generate` | 上传文件，生成本体定义 |
| POST | `/build` | 异步构建图谱 |
| GET | `/task/<task_id>` | 查询构建任务进度 |
| GET | `/data/<graph_id>` | 获取图谱数据（节点/边） |
| GET | `/project/<id>` | 获取项目详情 |
| GET | `/project/list` | 列出所有项目 |
| DELETE | `/project/<id>` | 删除项目 |
| POST | `/project/<id>/reset` | 重置项目状态 |
| DELETE | `/delete/<graph_id>` | 删除 Zep 图谱 |

### /api/simulation — 模拟管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/configure` | 配置模拟（生成人设和参数） |
| POST | `/start` | 启动模拟 |
| GET | `/<id>/status` | 查询模拟状态和进度 |
| POST | `/<id>/stop` | 停止模拟 |
| GET | `/<id>/actions` | 获取 Agent 行为记录 |
| GET | `/list` | 列出所有模拟 |

### /api/report — 报告管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/generate` | 异步生成报告 |
| POST | `/generate/status` | 查询生成进度 |
| GET | `/<id>` | 获取报告详情 |
| GET | `/<id>/sections` | 获取已生成章节 |
| GET | `/<id>/download` | 下载 Markdown 报告 |
| POST | `/chat` | 与 Report Agent 对话 |
| GET | `/<id>/agent-log` | 获取 Agent 执行日志 |
| GET | `/<id>/console-log` | 获取控制台日志 |
| GET | `/check/<sim_id>` | 检查报告状态 |

## 异步任务机制

图谱构建和报告生成都是耗时操作，采用后台线程 + 轮询模式：

```
客户端                     服务端
  │                          │
  │  POST /build             │
  │─────────────────────────►│ 创建 Task, 启动后台线程
  │  {"task_id": "xxx"}      │
  │◄─────────────────────────│
  │                          │
  │  GET /task/xxx (轮询)    │
  │─────────────────────────►│ 返回 progress/status
  │  {"progress": 45, ...}   │
  │◄─────────────────────────│
  │       ...重复轮询...      │
  │                          │
  │  GET /task/xxx           │
  │─────────────────────────►│
  │  {"status":"completed"}  │
  │◄─────────────────────────│
```

`TaskManager` 是线程安全的单例，内存中维护任务状态，支持创建、更新、查询。

## 前端页面路由

| 路径 | 组件 | 说明 |
|------|------|------|
| `/` | Home | 首页，项目列表，新建项目 |
| `/process/:projectId` | MainView | 主流程页面（Step 1-5 组件） |
| `/simulation/:id` | SimulationView | 模拟配置详情 |
| `/simulation/:id/start` | SimulationRunView | 模拟运行实时监控 |
| `/report/:id` | ReportView | 报告查看 |
| `/interaction/:id` | InteractionView | 深度交互对话 |

## 外部依赖说明

### Zep Cloud

Zep 提供 GraphRAG 能力，MiroFish 使用其 Standalone Graph 功能：
- 自动从文本中提取实体和关系
- 支持本体约束（限定实体和关系类型）
- 语义搜索 + 图遍历的混合检索
- 免费额度可支撑简单使用

### OASIS (CAMEL-AI)

OASIS 是开源的多 Agent 社交模拟框架：
- 模拟 Twitter / Reddit 两种社交平台
- 每个 Agent 有独立的 LLM 驱动行为
- 支持自定义动作集和行为逻辑
- MiroFish 通过 camel-oasis 和 camel-ai 包集成

### LLM API

通过 OpenAI SDK 格式调用，兼容：
- 阿里通义千问（推荐，百炼平台）
- DeepSeek
- OpenAI GPT 系列
- 任何兼容 OpenAI API 格式的服务

只需配置 `LLM_BASE_URL` 和 `LLM_API_KEY` 即可切换。
