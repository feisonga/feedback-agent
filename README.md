# Feedback Agent · 智能客服反馈系统

**规则优先、AI 兜底** — 80% 常见问题毫秒级规则回复，20% 疑难问题交 LLM 深度推理。

[![Python](https://img.shields.io/badge/Python-3.11-blue)]()
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-green)]()
[![React](https://img.shields.io/badge/React-18-61dafb)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()

---

## 🧠 核心架构：ReAct Agent

基于 **Yao et al. 2022 ReAct** 范式，LLM 自主完成 **Thought → Action → Observation** 推理循环，非预设流程。

```
用户消息
  │
  ▼
┌─────────────────────────────────────────────┐
│            ReAct Loop (最多 5 步)              │
│                                             │
│  Thought ──→ Action (调用工具)                │
│    ↑            │                           │
│    │            ▼                           │
│    └──── Observation (观察结果)               │
│                                             │
│  LLM 自主决定：调哪个工具 / 调几个 / 何时结束    │
└─────────────────────────────────────────────┘
  │
  ▼
最终回复 + 完整推理链路
```

**8 个 Agent 工具**（OpenAI Function Call 协议注册）：

| 工具 | 功能 | 驱动方式 |
|------|------|----------|
| `intent_classify` | 意图识别 (16 类) | LLM + 规则降级 |
| `sentiment_analyze` | 情感分析 | LLM + 词典降级 |
| `entity_extract` | 实体提取 | LLM + 正则降级 |
| `urgency_assess` | 紧急度评估 | LLM + 规则降级 |
| `faq_retrieve` | FAQ 语义检索 | sentence-transformers |
| `policy_search` | 政策文档匹配 | Whoosh BM25 |
| `similar_case_search` | 历史工单检索 | 向量相似度 |
| `dialogue_track` | 对话状态追踪 | 槽位填充 |

> **降级策略**：LLM 不可用时，全部工具自动回退到规则引擎（正则/词典/BM25），系统零崩溃。

---

## 🛠 技术栈

| 层 | 技术 |
|---|---|
| 后端框架 | Python 3.11 + FastAPI |
| ASGI 服务器 | Uvicorn (REST + WebSocket) |
| 数据库 | SQLite (aiosqlite) + SQLAlchemy 2.0 async |
| LLM | DeepSeek Chat API (OpenAI 兼容) |
| 向量模型 | sentence-transformers (paraphrase-multilingual-MiniLM-L12-v2) |
| 前端 | React 18 + Vite + Tailwind CSS 3 + TanStack Query 5 |
| 部署 | Docker Compose 一键启动 |

---

## 🚀 快速开始

### 1. 配置 API Key

```bash
cp env.example backend/.env
# 编辑 backend/.env，填入 DeepSeek API Key:
# OPENAI_API_KEY=***### 2. 安装依赖并启动

```bash
# 后端
cd feedback-agent
uv venv
uv pip install -e backend/
HF_HUB_OFFLINE=1 .venv/Scripts/python.exe -m uvicorn backend.app.main:app --port 3100

# 前端（新终端）
cd frontend
npm install
npm run dev
```

> 打开 **http://localhost:5173** 即可对话。

### Docker 一键启动

```bash
docker compose up -d
```

---

## 📡 API

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/health` | 健康检查 + Agent 加载状态 |
| `POST` | `/api/sessions` | 创建会话 |
| `POST` | `/api/chat/send` | 发送消息 (REST，返回 ReAct 链路) |
| `WS` | `/ws/chat/{session_id}` | 流式对话 (WebSocket) |
| `GET` | `/api/analytics/overview` | 分析概览 |

### 响应示例

```json
{
  "code": 0,
  "data": {
    "reply": "老乡，别着急！猪病倒了咱们得赶紧处理...",
    "intent": "emergency",
    "sentiment": "neutral",
    "agent_trace": {
      "steps": [
        {
          "step": 1,
          "thought": "我先分析一下您的情况...",
          "tool_calls": [
            {"name": "intent_classify", "arguments": {...}},
            {"name": "sentiment_analyze", "arguments": {...}},
            {"name": "urgency_assess", "arguments": {...}},
            {"name": "entity_extract", "arguments": {...}}
          ],
          "observations": [...]
        }
      ],
      "total_tool_calls": 4,
      "total_latency_ms": 14000
    }
  }
}
```

---

## 📁 项目结构

```
feedback-agent/
├── backend/                   # FastAPI 应用
│   ├── app/
│   │   ├── main.py            # 入口 + lifespan
│   │   ├── config.py          # Pydantic Settings + .env
│   │   ├── api/               # REST + WebSocket 路由
│   │   ├── core/              # ReAct 循环 · 工具注册 · 调度器
│   │   ├── models/            # SQLAlchemy ORM
│   │   ├── schemas/           # Pydantic Schema
│   │   └── services/          # 业务服务层
│   └── tests/
├── frontend/                  # React SPA
│   └── src/
│       ├── pages/             # ChatPage · AnalyticsPage
│       ├── components/        # ChatWindow · AgentTracePanel · Sidebar
│       ├── hooks/             # useChat · useChatStream · useAgentTrace
│       └── api/               # API client
├── python/                    # Agent 核心 + 知识库
│   ├── core/                  # ReAct 循环引擎 · Tool Registry · Memory
│   ├── agents/                # 8 个 Agent + LLM Tool 封装
│   └── knowledge/             # YAML/JSON/TXT 知识库 (20 FAQ + 3 政策文档)
├── docs/                      # 项目说明文档
├── docker-compose.yml
└── README.md
```

---

## 📖 设计理念

灵感来自王秋禾《增能视角下"社工+志愿者"协同模式》论文：

> 把 "社工 + 志愿者" 的分工逻辑搬到计算机系统里

| 论文路径 | 项目实现 |
|----------|----------|
| 优化资源配置 | 规则处理 80%，LLM 只处理 20% 疑难问题 |
| 机制创新 | 收到消息 → 判断 → 分配 → 回复，全程自动化 |
| 精准补位 | ReAct 调度中心决定"该谁管"，8 工具各管一摊 |
| 专业+协同 | 每个工具只做一件擅长的事，合起来就变聪明 |

---

## ⚖️ 与普通 AI 客服的差异

| | 普通 AI 客服 | 本项目 |
|---|---|---|
| 处理方式 | 每条消息都丢给 AI | 先走规则，搞不定再叫 AI |
| 回复速度 | 依赖 AI 响应 | 规则路径 <50ms |
| 成本 | 每条消息都烧 Token | 80% 消息零成本 |
| 可解释性 | 说不清为什么这样回 | 每条回复可追溯推理链路 |
| 确定性 | 同样问题可能回答不一样 | 规则判断每次都一样 |
