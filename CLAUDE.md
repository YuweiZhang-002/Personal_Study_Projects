# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Smart QoS-Scheduler** — A full-stack web app combining an LLM-powered anime character assistant with Pomodoro timer, task scheduling, PDF analysis, and campus/enterprise 5G QoS gateway requests.

## Development Commands

### Backend (Node.js + Express + TypeScript)
```bash
cd backend
npm install
npm run dev      # Development server on port 3001 (ts-node-dev with hot reload)
npm run build    # Compile TypeScript → dist/
npm start        # Run compiled output
```

### Frontend (React + Vite + TypeScript)
```bash
cd frontend
npm install
npm run dev      # Vite dev server on port 5173 (proxies /api → localhost:3001)
npm run build    # Type-check + bundle
npm run preview  # Preview production build
```

### Environment Setup
Copy `backend/.env.example` to `backend/.env` and fill in:
- `GEMINI_API_KEY` — required for all LLM features
- `GEMINI_MODEL` — defaults to `gemini-2.5-flash`
- `MAX_BANDWIDTH` — cap for QoS requests (e.g., `400`)
- `MOCK_QOS_REJECT=true` — simulates gateway rejection in dev

No linter or test runner is configured in this project.

## Architecture

### Backend (`backend/src/`)
Layered Express app:
- `index.ts` → `app.ts` (CORS, 10MB JSON limit, route mounting, error handlers)
- **Routes**: `routes/index.ts` mounts `/api/qos`, `/api/llm` (chat, stream, analyze)
- **Controllers**: validate input, format responses (`qos`, `chat`, `pdf`)
- **Services**:
  - `services/llm/gemini.service.ts` — Gemini API with function calling support
  - `services/llm/systemPrompt.ts` — context-aware prompt generation (timer/warning state)
  - `services/llm/tools.ts` — declares `show_qos_application_form` function tool
  - `services/qos/gateway.service.ts` — gateway HTTP request + mock rejection mode
  - `services/pdf/` — session-based analysis context (tracks chapter, iteration, history)
- **Utils**: `utils/jsonExtract.ts` — safe JSON parse with regex fallback for malformed LLM output
- **Middleware**: `AppError` class + global error handler returns `{ status, msg }`

### Frontend (`frontend/src/`)
All components are functional React with Tailwind CSS styling.

**State Management (Zustand stores):**
| Store | Manages |
|-------|---------|
| `useTimerStore` | Pomodoro phase, running/paused, minutes/seconds, durations |
| `useTaskStore` | Task list, add/delete, special task toggle |
| `useCharacterStore` | Emotion, dialogue bubble text, menu visibility, `tenMinWarning` flag |
| `useQosStore` | QoS modal open/close state |
| `useChatStore` | Chat panel visibility |
| `usePdfStore` | PDF panel visibility |
| `useSettingsStore` | Volume, analysis depth, character image |

**Key Components:**
| Component | Role |
|-----------|------|
| `VirtualRoom` | Three-column flex layout: (timer+tasks) | (character) | (notebook+settings) |
| `AnimeCharacter` | Emotion-based image, popup menu (chat/PDF/QoS), dialogue bubbles |
| `PomodoroTimer` | Circular countdown, focus/break cycle |
| `TaskList` | Accordion list with special task toggle |
| `QosModal` | Bandwidth-validated form submission |
| `ChatPanel` | Chat UI with streaming LLM, intercepts function call responses |
| `PdfAnalysisPanel` | Upload → chunk → iterative analysis loop with "understood"/"explain more" flow |

**Hooks:**
- `useTaskPolling` — polls every 60s, detects <10 min to special task, sets `tenMinWarning`, stops timer if running
- `useTimerCountdown` — drives timer tick via `setInterval`

**API layer** (`src/api/*.ts`): thin fetch wrappers for `sendChat`, `requestQos`, `analyzePdf`

### Key Cross-Cutting Patterns

**LLM Function Calling:** The AI can invoke `show_qos_application_form(reason, recommended_bandwidth)`. The backend returns `{ functionCall }` in the response; the frontend detects this and opens QosModal automatically.

**Context-Aware System Prompt:** `buildSystemPrompt()` injects timer and warning state. When `isTimerRunning=true`, the character refuses casual chat. When `isTenMinWarning=true`, the prompt instructs the AI to call the QoS tool.

**Emotion-Driven Character UI:** Backend includes an `emotion` field (`normal | happy | thinking | strict | worried`) in all LLM responses. Maps to `/public/assets/character/{emotion}.jpg`.

**Safe JSON Extraction:** LLM responses are parsed with a try/catch + regex fallback (`/\{[\s\S]*\}/`) to handle markdown-wrapped or partial JSON. Located in both `backend/src/utils/jsonExtract.ts` and `frontend/src/utils/jsonExtract.ts`.

**Session-Based PDF Analysis:** Each analysis session has a `sessionId`. The backend tracks current chunk index, iteration count, and recent conversation history server-side. User clicks "understood" or "explain more" to advance.

**QoS Bandwidth Cap:** Frontend enforces `MAX_BANDWIDTH` before submission. Backend also clamps the value and forwards to the gateway. `Req_Fail` responses from the gateway surface as character dialogue bubbles.

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/qos/request` | Submit QoS network resource request |
| `POST` | `/api/llm/chat` | Chat message → `{ text, emotion, functionCall }` |
| `POST` | `/api/llm/chat/stream` | SSE streaming variant of chat |
| `POST` | `/api/llm/analyze` | PDF chunk analysis → `{ sessionId, script[], emotion, functionCall }` |
| `GET`  | `/health` | Returns `{ status: "ok", timestamp }` |

## Path Alias

Frontend TypeScript uses `@/*` → `src/*` (configured in `tsconfig.json` and `vite.config.ts`).
