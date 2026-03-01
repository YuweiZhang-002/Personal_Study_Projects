# Smart QoS-Scheduler 项目结构说明

> 本文档详细描述项目中每个模块对应的文件位置，以及各功能的实现与修改过程。
>
> 每个源文件顶部已添加模块注释，格式为：`模块：XXX`、`文件：path/to/file`

---

## 一、项目总览

```
APP_LLM/
├── Readme.md                 # 需求文档
├── PROJECT_STRUCTURE.md      # 本文档：项目结构与功能说明
├── backend/                  # 后端 (Node + Express)
└── frontend/                 # 前端 (React + Zustand)
```

---

## 二、模块与文件映射表

### 2.1 后端 (Backend)

| 模块 | 功能 | 对应文件 |
|------|------|----------|
| **入口** | Express 服务启动 | `backend/src/index.ts` |
| **应用配置** | CORS、JSON 解析、路由挂载、错误处理 | `backend/src/app.ts` |
| **环境变量** | 读取 PORT、GEMINI_API_KEY、MAX_BANDWIDTH 等 | `backend/src/config/env.ts` |
| **错误处理** | AppError 类、全局错误中间件、404 处理 | `backend/src/middleware/errorHandler.ts` |
| **路由聚合** | 挂载 /api/qos、/api/llm 子路由 | `backend/src/routes/index.ts` |
| **QoS 路由** | POST /api/qos/request | `backend/src/routes/qos.routes.ts` |
| **聊天路由** | POST /api/llm/chat、/api/llm/chat/stream | `backend/src/routes/chat.routes.ts` |
| **PDF 路由** | POST /api/llm/analyze | `backend/src/routes/pdf.routes.ts` |
| **QoS 控制器** | 校验请求、调用网关、返回 success/Req_Fail | `backend/src/controllers/qos.controller.ts` |
| **聊天控制器** | 调用 Gemini、处理流式/非流式 | `backend/src/controllers/chat.controller.ts` |
| **PDF 控制器** | 接收 PDF 切片、调用分析服务 | `backend/src/controllers/pdf.controller.ts` |
| **Gemini 服务** | 封装 @google/genai、chat/chatStream | `backend/src/services/llm/gemini.service.ts` |
| **系统 Prompt** | 时雨角色设定、isTimerRunning/isTenMinWarning 逻辑 | `backend/src/services/llm/systemPrompt.ts` |
| **Function 工具** | show_qos_application_form 定义 | `backend/src/services/llm/tools.ts` |
| **PDF 分析服务** | Galgame 讲解、分段状态机、Function Call | `backend/src/services/llm/pdfAnalysis.service.ts` |
| **PDF Prompt** | 文献讲解 Prompt 模板 | `backend/src/services/llm/pdfPrompt.ts` |
| **分析上下文** | Session 存储、chunks/currentIndex/history | `backend/src/services/pdf/analysisContext.ts` |
| **QoS 网关** | 调用 GATEWAY_URL 或 mock 返回 | `backend/src/services/qos/gateway.service.ts` |
| **JSON 安全解析** | 正则二次提取、extractEmotion | `backend/src/utils/jsonExtract.ts` |
| **类型定义** | QosRequestBody、ChatRequestBody、PdfAnalysisRequestBody | `backend/src/types/*.ts` |

### 2.2 前端 (Frontend)

| 模块 | 功能 | 对应文件 |
|------|------|----------|
| **入口** | React 挂载、全局样式 | `frontend/src/main.tsx` |
| **根组件** | 挂载 VirtualRoom、弹窗、任务轮询 | `frontend/src/App.tsx` |
| **虚拟房间** | 主布局、承载各子模块 | `frontend/src/components/VirtualRoom/VirtualRoom.tsx` |
| **虚拟角色** | 角色图、菜单、对话气泡、10 分钟提醒选项 | `frontend/src/components/AnimeCharacter/AnimeCharacter.tsx` |
| **番茄钟** | 倒计时、开始/暂停/终止、专注-休息循环 | `frontend/src/components/PomodoroTimer/PomodoroTimer.tsx` |
| **任务清单** | 手风琴、添加/删除任务、特殊任务 Toggle | `frontend/src/components/TaskList/TaskList.tsx` |
| **备忘录** | 新建/删除页面、标题与内容、localStorage 持久化 | `frontend/src/components/Notebook/Notebook.tsx` |
| **设置** | 闹铃音量、分析深度、角色图说明 | `frontend/src/components/Settings/Settings.tsx` |
| **QoS 弹窗** | 表单、提交、Req_Fail 拟人化、成功后 LLM 鼓励 | `frontend/src/components/QosModal/QosModal.tsx` |
| **聊天面板** | 日常对话、LLM 调用、Function Call 打开 QoS | `frontend/src/components/ChatPanel/ChatPanel.tsx` |
| **PDF 分析面板** | 上传 PDF、切片、讲解、听懂了/再解释、QoS 强插 | `frontend/src/components/PdfAnalysisPanel/PdfAnalysisPanel.tsx` |
| **任务轮询** | 每分钟检查、10 分钟提醒、强制终止番茄钟 | `frontend/src/hooks/useTaskPolling.ts` |
| **番茄钟倒计时** | setInterval 驱动 tick | `frontend/src/hooks/useTimerCountdown.ts` |
| **Zustand 状态** | useTimerStore、useTaskStore、useCharacterStore 等 | `frontend/src/stores/*.ts` |
| **API 调用** | sendChat、requestQos、analyzePdf | `frontend/src/api/*.ts` |
| **工具函数** | pdfChunking、jsonExtract | `frontend/src/utils/*.ts` |

---

## 三、功能实现与修改过程

### 3.1 后端模块

#### 3.1.1 环境变量读取 (`backend/src/config/env.ts`)

- **功能**：通过 `dotenv` 加载 `.env`，导出 `env` 对象（PORT、GEMINI_API_KEY、MAX_BANDWIDTH、GATEWAY_URL、GEMINI_MODEL、MOCK_QOS_REJECT）
- **修改过程**：初始仅 NODE_ENV/PORT/GEMINI_API_KEY；后续增加 MAX_BANDWIDTH、GATEWAY_URL、GEMINI_MODEL、MOCK_QOS_REJECT；`validateEnv()` 用于启动时检查必填项

#### 3.1.2 错误处理中间件 (`backend/src/middleware/errorHandler.ts`)

- **功能**：`AppError` 自定义错误类；`errorHandler` 统一返回 JSON `{ status, msg }`；`notFoundHandler` 处理 404
- **修改过程**：骨架阶段定义 AppError 与 errorHandler；保持与 Readme 中 Req_Fail 等错误码一致

#### 3.1.3 QoS 申请 (`POST /api/qos/request`)

- **涉及文件**：`qos.routes.ts` → `qos.controller.ts` → `gateway.service.ts`、`qos.types.ts`
- **流程**：校验 student_id/org_code/network_params → 带宽限制 MAX_BANDWIDTH → 调用 `requestGatewayQos` → 成功 200 / 失败 403 Req_Fail
- **修改过程**：骨架阶段仅做格式校验；完善后增加 `gateway.service.ts`，支持 GATEWAY_URL 真实调用与 MOCK_QOS_REJECT 模拟拒绝

#### 3.1.4 LLM 日常聊天 (`POST /api/llm/chat`)

- **涉及文件**：`chat.routes.ts` → `chat.controller.ts` → `gemini.service.ts`、`systemPrompt.ts`、`tools.ts`、`chat.types.ts`
- **流程**：接收 message/history/isTimerRunning/isTenMinWarning → 构建 System Instruction → 调用 Gemini → 返回 text、emotion、functionCall
- **修改过程**：骨架阶段返回占位；完善后集成 @google/genai，实现 System Prompt、Function Calling（show_qos_application_form）、emotion 提取

#### 3.1.5 LLM 流式聊天 (`POST /api/llm/chat/stream`)

- **涉及文件**：同上，`chat.controller.ts` 中 `handleChatStream`
- **流程**：设置 SSE 头 → `generateContentStream` → 按 chunk 写入 `data: {...}\n\n`
- **修改过程**：与 chat 同时实现，用于前端流式展示（当前前端 ChatPanel 使用非流式）

#### 3.1.6 PDF 文献分析 (`POST /api/llm/analyze`)

- **涉及文件**：`pdf.routes.ts` → `pdf.controller.ts` → `pdfAnalysis.service.ts`、`analysisContext.ts`、`pdfPrompt.ts`、`pdf.types.ts`
- **流程**：接收 chunks/sessionId/userFeedback/analysisDepth/isTenMinWarning → 从 analysisContext 取/建 Session → 调用 Gemini → 解析 script 数组 → 返回 sessionId、script、emotion、functionCall
- **修改过程**：新增模块；实现 analysis_context 状态机、Galgame 风格 Prompt、JSON 安全解析、QoS 强插逻辑

#### 3.1.7 JSON 安全解析 (`backend/src/utils/jsonExtract.ts`)

- **功能**：`safeParseJson` 使用正则 `/\{[\s\S]*\}/` 二次提取；`extractEmotion` 从 LLM 文本中解析 emotion
- **修改过程**：按 Readme Safety Check 要求实现，防止 LLM 多输出导致解析崩溃

---

### 3.2 前端模块

#### 3.2.1 VirtualRoom 主容器 (`frontend/src/components/VirtualRoom/VirtualRoom.tsx`)

- **功能**：虚拟房间布局，左侧 PomodoroTimer + TaskList，中间 AnimeCharacter，右侧 Notebook + Settings
- **修改过程**：骨架阶段搭建基础 Flex 布局；后续保持结构不变

#### 3.2.2 AnimeCharacter 虚拟角色 (`frontend/src/components/AnimeCharacter/AnimeCharacter.tsx`)

- **功能**：角色图（按 emotion 切换）、点击打开菜单（日常聊天/报告分析/QoS 申请）、对话气泡、10 分钟提醒时「申请专属网络」「不需要」
- **修改过程**：骨架阶段仅占位；完善后接入 useChatStore、usePdfStore、useQosStore，实现菜单跳转；增加 tenMinWarning 气泡与按钮；角色图支持 `/assets/character/*.jpg` 与加载失败占位

#### 3.2.3 PomodoroTimer 番茄钟 (`frontend/src/components/PomodoroTimer/PomodoroTimer.tsx`)

- **功能**：圆形倒计时、开始/暂停/终止、专注-休息循环、可配置专注/休息时长
- **修改过程**：骨架阶段仅按钮；完善后接入 `useTimerCountdown`，实现 `tick` 驱动；增加 focusDuration/breakDuration 配置；phase 切换 focus/break

#### 3.2.4 TaskList 任务清单 (`frontend/src/components/TaskList/TaskList.tsx`)

- **功能**：手风琴展开/收起、添加任务（名称、时间、特殊任务）、删除、显示计划时间
- **修改过程**：骨架阶段仅展示；完善后增加添加表单、datetime-local、删除按钮、formatTime 显示

#### 3.2.5 useTaskPolling 任务轮询 (`frontend/src/hooks/useTaskPolling.ts`)

- **功能**：每分钟检查任务时间，距开始 <10 分钟时：设置 tenMinWarning、显示气泡、若番茄钟运行则强制 stop
- **修改过程**：新增 Hook；在 App 中调用 `useTaskPolling()`；与 useCharacterStore、useTimerStore 联动

#### 3.2.6 ChatPanel 日常聊天 (`frontend/src/components/ChatPanel/ChatPanel.tsx`)

- **功能**：消息列表、输入框、调用 sendChat；收到 functionCall 时 openQosModal；传递 isTimerRunning、isTenMinWarning
- **修改过程**：新增组件；接入 useChatStore、useCharacterStore、useQosStore；实现 Function Call 分发

#### 3.2.7 QosModal QoS 申请弹窗 (`frontend/src/components/QosModal/QosModal.tsx`)

- **功能**：表单（学号、组织编号、IP、端口、时长、带宽）、提交 requestQos；成功时调用 sendChat 获取鼓励回复；失败时 setBubble 拟人化提示
- **修改过程**：骨架阶段仅占位表单；完善后实现完整字段、带宽限制、提交逻辑、Req_Fail 反馈、成功后 LLM 鼓励流程

#### 3.2.8 PdfAnalysisPanel 报告分析 (`frontend/src/components/PdfAnalysisPanel/PdfAnalysisPanel.tsx`)

- **功能**：上传 PDF、pdf.js 解析、chunkText 切片、调用 analyzePdf；展示 script；「听懂了」「这里再解释下」；QoS 强插时「继续」「终止」
- **修改过程**：新增组件；集成 pdfjs-dist、chunkText；实现 analysisDepth 从 Settings 读取；处理 functionCall 打开 QoS、qosInterrupted 状态

#### 3.2.9 Notebook 备忘录 (`frontend/src/components/Notebook/Notebook.tsx`)

- **功能**：新建/删除页面、标题与内容编辑、localStorage 持久化（STORAGE_KEY）
- **修改过程**：骨架阶段占位；完善后实现 loadNotes/saveNotes、addNote、removeNote、updateNote

#### 3.2.10 Settings 设置 (`frontend/src/components/Settings/Settings.tsx`)

- **功能**：闹铃音量滑块、分析深度下拉（10/20/30/40/50）、角色图路径说明
- **修改过程**：骨架阶段已有音量与深度；完善后增加角色图说明（API Key 不暴露，按 Readme 要求）

---

## 四、数据流与模块依赖

```
┌─────────────────────────────────────────────────────────────────┐
│                          App.tsx                                 │
│  useTaskPolling → VirtualRoom + QosModal + ChatPanel + PdfPanel  │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ useTimerStore│    │useCharacterStore│   │ useQosStore  │
│ useTaskStore │    │ useChatStore   │   │ usePdfStore  │
└──────────────┘    └──────────────┘    └──────────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              ▼
                    ┌──────────────────┐
                    │  API: chat, qos, │
                    │  pdf (→ backend) │
                    └──────────────────┘
```

---

## 五、配置文件说明

| 文件 | 用途 |
|------|------|
| `backend/package.json` | 后端依赖：express、@google/genai、cors、dotenv |
| `backend/tsconfig.json` | 后端 TS 编译配置 |
| `backend/.env.example` | 环境变量模板 |
| `frontend/package.json` | 前端依赖：react、zustand、pdfjs-dist、lucide-react |
| `frontend/vite.config.ts` | Vite 配置、/api 代理到 3001 |
| `frontend/tsconfig.json` | 前端 TS 配置 |
| `frontend/tailwind.config.js` | Tailwind 配置 |
| `frontend/postcss.config.js` | PostCSS 配置 |

---

## 六、启动方式

```bash
# 后端
cd backend && npm install && npm run dev

# 前端
cd frontend && npm install && npm run dev
```

后端默认 `http://localhost:3001`，前端默认 `http://localhost:5173`，前端通过 Vite 代理将 `/api` 转发到后端。
