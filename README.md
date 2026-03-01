# APP_LLM-Based Pomodoro Clock

<div align="center">

![React](https://img.shields.io/badge/React-18.3-61DAFB?logo=react&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-5.6-3178C6?logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-5.4-646CFF?logo=vite&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-Express-339933?logo=node.js&logoColor=white)
![Gemini](https://img.shields.io/badge/Gemini-2.5_Flash-4285F4?logo=google&logoColor=white)
![Zustand](https://img.shields.io/badge/Zustand-4.5-FF6B35)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-3.4-06B6D4?logo=tailwindcss&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

**一个将 5G 网络 QoS 调度与 LLM 虚拟角色融合的智能工作站**

A smart workstation fusing 5G network QoS scheduling with an LLM-powered anime companion.

[核心功能](#-核心功能--features) · [技术架构](#-技术架构--architecture) · [快速启动](#-快速启动--quick-start) · [环境变量](#-环境变量配置--environment-variables) · [API 文档](#-api-接口文档--api-reference) · [角色系统](#-角色表情系统--character-emotion-system)

</div>

---

## ✨ 项目简介 · Overview

**LLM-Based Pomodoro Clock** 是一个面向校园 / 企业场景的智能工作站。它将番茄钟、任务调度、5G QoS 网络申请与 LLM 虚拟角色（时雨）整合在同一个沉浸式虚拟房间界面中。

**LLM-Based Pomodoro Clock** is a campus/enterprise smart workstation that integrates a Pomodoro timer, task scheduling, 5G QoS network requests, and an LLM virtual character (Shiyu) into a single immersive virtual room interface.

核心理念：**让 AI 助手感知你的工作状态，并在合适的时机主动帮你申请专属网络资源。**

Core idea: **Let the AI assistant sense your work state and proactively request dedicated network resources at the right moment.**

- 💬 **对话感知 / Context-aware chat**：LLM 读取番茄钟状态，专注时拒绝闲聊，休息时才开放日常对话
- 📡 **自动 QoS 触发 / Auto QoS trigger**：任务开始前 10 分钟，角色主动弹出 QoS 申请表单（Function Calling 驱动）
- 📄 **报告分析**：将枯燥的学术 PDF 转化为文字风格的分段对话式讲解 / Transforms academic PDFs into interactive visual-novel-style explanations
- 🎭 **情绪驱动 UI / Emotion-driven UI**：角色图像随 LLM 返回的 `emotion` 字段实时切换（`normal` / `happy` / `thinking` / `strict` / `worried`）

---

## 📁 项目结构 · Project Structure

```
smart-qos-scheduler/
├── backend/                  # Express + TypeScript API server
│   ├── src/
│   │   ├── services/llm/     # Gemini integration, System Prompt, Function Tools
│   │   ├── services/qos/     # QoS gateway communication
│   │   ├── controllers/      # Request validation & response formatting
│   │   └── utils/            # Safe JSON parser for malformed LLM output
│   └── .env.example
└── frontend/                 # React SPA
    └── src/
        ├── components/       # VirtualRoom, AnimeCharacter, PomodoroTimer...
        ├── stores/           # 7 Zustand stores
        ├── hooks/            # useTaskPolling, useTimerCountdown
        ├── api/              # Fetch wrappers (chat / pdf / qos)
        └── utils/            # pdfChunking, jsonExtract
```
(注：frontend部分内容需要额外创建文件夹并且迁移对应的文件 / Note: contexts in the frontend are not zipped in the file, creating a file named "frontend" is needed)
---

## 🚀 核心功能 · Features

### 🍅 番茄钟 · Pomodoro Timer

```
idle ──[Start]──► focus ──[Time's up]──► break ──[Time's up]──► focus (loop)
         │                                │
      [Stop]                           [Stop]
         └──────────────────────────────► idle
```

- 可调节专注 / 休息时长，圆形倒计时 UI / Adjustable focus/break durations with a circular countdown UI
- 计时期间向 LLM 注入 `isTimerRunning: true`，角色切换为 `strict` 表情并屏蔽闲聊 / Injects `isTimerRunning: true` into the LLM context during focus phase, switching the character to `strict` mode and blocking casual chat

### 📋 任务调度 · Task Scheduling — 十分钟抢占 / 10-Minute Pre-emption

- 任务设置"特殊任务"开关后，`useTaskPolling` 每分钟轮询系统时间 / After marking a task as "special", `useTaskPolling` checks the system clock every minute
- 距任务开始 ≤ 10 分钟时 / When ≤ 10 minutes remain before a special task:
  1. 角色气泡弹出提醒，触发 `tenMinWarning` 标志 / Character bubble fires, setting the `tenMinWarning` flag
  2. 若番茄钟正在运行，强制终止计时（抢占机制）/ If Pomodoro is running, the timer is forcibly stopped (pre-emption)
  3. 额外显示 [申请专属网络] / [不需要] 选项 / Extra options appear: **Apply for dedicated network** / **Not needed**

### 🤖 LLM 对话 · Function Calling

`ChatPanel` 与 `PdfAnalysisPanel` 均支持 Gemini Function Calling / Both panels support Gemini Function Calling:

| Tool Name | 触发条件 / Trigger | 前端响应 / Frontend Action |
|-----------|----------|----------|
| `show_qos_application_form` | AI 判断用户需要 QoS 服务 / AI determines QoS is needed | 自动打开 QosModal 并预填带宽建议值 / Auto-opens QosModal with pre-filled bandwidth |

### 📄 PDF 分析 · PDF Analysis

1. 前端用 `pdf.js` 提取 PDF 全文，按段落切片（默认 1500 字/块）/ Frontend extracts full text with `pdf.js` and slices it into ~1500-char chunks
2. 按"分析深度"（10–50 次迭代）逐块发送给后端 LLM / Sends chunks to the backend LLM across 10–50 iterations based on the configured analysis depth
3. LLM 以角色口吻讲故事式解读，用户点击 / LLM interprets content in character voice; the user clicks:
   - **听懂了 / Understood** → 进入下一块 / Advance to next chunk
   - **再解释下 / Explain more** → 对当前块深化分析 / Deepen analysis on the current chunk
4. 后端维护 `sessionId` 保持跨请求上下文 / Backend maintains `sessionId` for cross-request context

### 📡 QoS 网关申请 · QoS Gateway Request

- 表单收集：工号 / 学号、组织编号、目标 IP/端口、时长、带宽 / Form collects employee/student ID, org code, target IP/port, duration, and bandwidth
- 前端强制校验带宽上限（`MAX_BANDWIDTH`），超限自动截断 / Frontend enforces `MAX_BANDWIDTH`; values above the cap are clamped before submission
- 网关返回 `Req_Fail` 时，角色气泡展示个性化错误提示 / On `Req_Fail` from the gateway, the character bubble displays a contextual error message

---

## 🏗 技术架构 · Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Browser (SPA)                     │
│  React 18 + TypeScript + Vite + Tailwind CSS        │
│                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐│
│  │  Zustand │ │ pdf.js   │ │    lucide-react       ││
│  │  7 stores│ │ chunking │ │    icons              ││
│  └──────────┘ └──────────┘ └──────────────────────┘│
│                     │ /api proxy                   │
└─────────────────────┼───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│              Express Backend (Node.js)              │
│  ┌─────────────────┐  ┌──────────────────────────┐  │
│  │  /api/llm/chat  │  │  /api/llm/analyze (PDF)  │  │
│  │  /api/qos/      │  │  Google Gemini 2.5 Flash  │  │
│  │  request        │  │  Function Calling         │  │
│  └─────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                      │
        ┌─────────────┴──────────────┐
        ▼                            ▼
  Google Gemini API           5G QoS Gateway
  (LLM + Function Call)       (GATEWAY_URL)
```

### 前端技术栈 · Frontend Stack

| 层次 / Layer | 技术 / Tech | 用途 / Purpose |
|------|------|------|
| UI 框架 | React 18 + TypeScript | 函数式组件，严格类型 / Functional components, strict typing |
| 构建工具 | Vite 5 | 开发热更新，生产打包 / HMR dev server, production bundling |
| 样式 | Tailwind CSS 3 | 原子化 CSS / Utility-first CSS |
| 状态管理 | Zustand 4 | 7 个独立 Store，跨组件共享 / 7 independent stores |
| PDF 解析 | pdfjs-dist 4 | 浏览器端提取 PDF 文本 / In-browser PDF text extraction |
| 图标 | lucide-react | SVG 图标库 / SVG icon library |

### 后端技术栈 · Backend Stack

| 层次 / Layer | 技术 / Tech | 用途 / Purpose |
|------|------|------|
| 运行时 | Node.js + Express 4 | REST API 服务 / REST API server |
| 语言 | TypeScript 5 | 全端类型一致 / Full-stack type consistency |
| LLM SDK | @google/genai 1.0 | Gemini API + Function Calling |
| 开发 | ts-node-dev | 热重载 / Hot reload |

---

## ⚡ 快速启动 · Quick Start

### 前置条件 · Prerequisites

- Node.js ≥ 18
- Google Gemini API Key（[获取地址 / Get key](https://aistudio.google.com/app/apikey)）

### 1. 克隆仓库 / Clone

```bash
git clone https://github.com/YuweiZhang-002/smart-qos-scheduler.git
cd smart-qos-scheduler
```

### 2. 配置后端环境变量 / Configure Backend Environment

```bash
cd backend
cp .env.example .env
# 编辑 .env，至少填写 GEMINI_API_KEY
# Edit .env — at minimum, set GEMINI_API_KEY
```

### 3. 启动后端 / Start Backend

```bash
# 在 backend/ 目录 / Inside backend/
npm install
npm run dev        # http://localhost:3001
```

### 4. 启动前端 / Start Frontend

```bash
# 新开终端，在 frontend/ 目录 / Open a new terminal inside frontend/
cd ../frontend
npm install
npm run dev        # http://localhost:5173
```

打开浏览器访问 **http://localhost:5173**，前端会自动将 `/api/*` 请求代理至后端。

Open **http://localhost:5173** in your browser — the frontend automatically proxies all `/api/*` requests to the backend.

---

## ⚙ 环境变量配置 · Environment Variables

所有配置项位于 `backend/.env`，前端**无需**任何密钥配置。

All variables live in `backend/.env`. The frontend requires **no** secrets.

| 变量名 / Variable | 必填 / Required | 默认值 / Default | 说明 / Description |
|--------|------|--------|------|
| `GEMINI_API_KEY` | ✅ | — | Google Gemini API 密钥 / API key |
| `GEMINI_MODEL` | | `gemini-2.5-flash` | 使用的模型 / Model ID |
| `PORT` | | `3001` | 后端监听端口 / Backend listen port |
| `MAX_BANDWIDTH` | | `100` | QoS 带宽上限（Mbps）/ Bandwidth cap for QoS requests |
| `GATEWAY_URL` | | — | 5G 网关地址（留空则跳过转发）/ 5G gateway URL (skip real forwarding if empty) |
| `MOCK_QOS_REJECT` | | `false` | 设为 `true` 模拟网关拒绝 / Set to `true` to simulate gateway rejection |
| `NODE_ENV` | | `development` | 生产模式隐藏错误详情 / Hides error details in production |

---


## 🎭 角色表情系统 · Character Emotion System

LLM 在每次响应中返回 `emotion` 字段，前端映射至对应角色图像。系统状态的优先级高于 LLM 返回值。

The LLM includes an `emotion` field in every response; the frontend maps it to the corresponding character image. System state takes priority over the LLM-returned value.

优先级（高→低）/ Override priority (high → low): `tenMinWarning` → `isTimerRunning` → LLM `emotion`

| `emotion` | 触发场景 / Trigger |
|---------|----------|
| `normal` | 默认待机 / Default idle state |
| `happy` | 聊天愉快、QoS 申请成功 / Pleasant chat, QoS approved |
| `thinking` | 分析 PDF、思考问题 / Analyzing PDF, pondering a question |
| `strict` | 番茄钟专注期间 / During Pomodoro focus phase |
| `worried` | QoS 申请被拒（Req_Fail）/ QoS request rejected |

### 替换角色图像 · Replacing Character Images

角色图像存放于 `frontend/public/assets/character/`，共 5 张 `.png` 文件：

Character images live in `frontend/public/assets/character/` — five `.png` files:

```
normal.png  happy.png  thinking.png  strict.png  worried.png
```

直接替换文件后，如果页面图像未更新，原因是浏览器缓存了旧图片（URL 未变化）。执行**强制刷新（Ctrl+Shift+R）**即可看到新图像。

After replacing a file in place, if the page still shows the old image, the browser has cached the previous version (the URL is unchanged). Do a **hard refresh (Ctrl+Shift+R)** to bypass the cache.

若需彻底避免缓存问题，可将图片移入 `frontend/src/assets/character/` 并通过 JS `import` 引入，Vite 会对文件内容生成哈希，替换文件后哈希自动更新，浏览器无需手动刷新：

To eliminate the cache issue entirely, move the images into `frontend/src/assets/character/` and import them directly. Vite will content-hash the filenames, so any file change automatically invalidates the browser cache:

```tsx
import normalImg   from '@/assets/character/normal.png';
import happyImg    from '@/assets/character/happy.png';
import thinkingImg from '@/assets/character/thinking.png';
import strictImg   from '@/assets/character/strict.png';
import worriedImg  from '@/assets/character/worried.png';

const EMOTION_IMAGES: Record<string, string> = {
  normal: normalImg, happy: happyImg, thinking: thinkingImg,
  strict: strictImg, worried: worriedImg,
};
```

