# MiroFish 安装部署指南

## 环境要求

| 工具 | 版本 | 说明 |
|------|------|------|
| Node.js | 18+ | 前端运行时，自带 npm |
| Python | ≥3.11, ≤3.12 | 后端运行时 |
| uv | 最新 | Python 包管理器，可自动下载指定版本 Python |

## 源码部署

### 1. 克隆仓库

```bash
git clone git@github.com:killinux/MiroFish.git
cd MiroFish
```

### 2. 配置环境变量

```bash
cp .env.example .env
```

编辑 `.env`，填入以下必填项：

```env
# LLM API（支持 OpenAI SDK 格式的任意 LLM API）
# 推荐阿里百炼平台 qwen-plus：https://bailian.console.aliyun.com/
LLM_API_KEY=your_api_key
LLM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_MODEL_NAME=qwen-plus

# Zep Cloud（免费额度即可）：https://app.getzep.com/
ZEP_API_KEY=your_zep_api_key
```

可选的加速 LLM 配置（不使用则不要写入 .env）：

```env
LLM_BOOST_API_KEY=...
LLM_BOOST_BASE_URL=...
LLM_BOOST_MODEL_NAME=...
```

### 3. 安装依赖

一键安装：

```bash
npm run setup:all
```

或分步安装：

```bash
# 前端依赖（根目录 + frontend/）
npm run setup

# 后端依赖（自动创建 .venv 虚拟环境）
npm run setup:backend
```

后端使用 `uv sync`，会自动下载符合要求的 Python 版本并创建虚拟环境。如果系统 Python 版本低于 3.11，可以先手动安装：

```bash
uv python install 3.12
```

### 4. 启动服务

```bash
# 前后端同时启动
npm run dev
```

启动后：
- 前端：http://localhost:3000
- 后端 API：http://localhost:5001
- 健康检查：http://localhost:5001/health

单独启动：

```bash
npm run backend    # 仅后端
npm run frontend   # 仅前端
```

### 5. 验证

```bash
# 后端健康检查
curl http://localhost:5001/health
# 返回 {"service":"MiroFish Backend","status":"ok"}

# 前端页面
curl -s http://localhost:3000/ | head -3
```

## Docker 部署

```bash
cp .env.example .env
# 编辑 .env 填入 API Key

docker compose up -d
```

端口映射：3000（前端）/ 5001（后端），从根目录 `.env` 读取配置。

## macOS 特殊说明

macOS 系统自带的 Python 通常是 3.9，不满足 >=3.11 的要求。解决方法：

```bash
# uv 会自动下载 Python 3.12 并在 .venv 中使用
uv python install 3.12
cd backend && uv sync
```

无需手动安装 Python，uv 会管理独立的 Python 版本。

## 配置说明

所有配置通过 `backend/app/config.py` 的 `Config` 类管理，从项目根目录 `.env` 加载：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `FLASK_HOST` | 0.0.0.0 | 后端监听地址 |
| `FLASK_PORT` | 5001 | 后端端口 |
| `FLASK_DEBUG` | True | 调试模式 |
| `LLM_API_KEY` | 无 | LLM API 密钥（必填） |
| `LLM_BASE_URL` | https://api.openai.com/v1 | LLM API 地址 |
| `LLM_MODEL_NAME` | gpt-4o-mini | LLM 模型名称 |
| `ZEP_API_KEY` | 无 | Zep Cloud 密钥（必填） |
| `OASIS_DEFAULT_MAX_ROUNDS` | 10 | 模拟默认最大轮数 |
| `REPORT_AGENT_MAX_TOOL_CALLS` | 5 | Report Agent 最大工具调用次数 |
| `REPORT_AGENT_MAX_REFLECTION_ROUNDS` | 2 | Report Agent 最大反思轮数 |
| `REPORT_AGENT_TEMPERATURE` | 0.5 | Report Agent 温度参数 |

## 目录结构

```
MiroFish/
├── .env.example          # 环境变量模板
├── .env                  # 实际配置（不入库）
├── package.json          # 根级 npm 脚本（dev/setup/build）
├── docker-compose.yml
├── Dockerfile
├── docs/                 # 文档
├── backend/
│   ├── pyproject.toml    # Python 项目定义与依赖
│   ├── uv.lock           # 锁定文件
│   ├── run.py            # 后端启动入口
│   └── app/
│       ├── __init__.py   # Flask 应用工厂
│       ├── config.py     # 配置管理
│       ├── api/          # REST 路由（graph / simulation / report）
│       ├── services/     # 核心业务逻辑（14 个模块）
│       ├── models/       # 数据模型
│       └── utils/        # 工具类
└── frontend/
    ├── package.json
    ├── index.html
    └── src/
        ├── App.vue
        ├── main.js
        ├── router/       # Vue Router
        ├── views/        # 页面组件
        ├── components/   # 步骤组件
        ├── api/          # 前端 API 封装
        └── i18n/         # 国际化
```
