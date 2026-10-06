# TraceNova — System Architecture Document

**Version:** 1.0  
**Status:** Draft  
**Last Updated:** March 17, 2026  
**Author:** TraceNova Engineering Team

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [High-Level System Diagram](#2-high-level-system-diagram)
3. [Frontend Architecture](#3-frontend-architecture)
4. [Backend Architecture](#4-backend-architecture)
5. [AI / LLM Layer](#5-ai--llm-layer)
6. [Data Flow & Sequence Diagrams](#6-data-flow--sequence-diagrams)
7. [File Upload Pipeline](#7-file-upload-pipeline)
8. [API Contract](#8-api-contract)
9. [Prompt Engineering](#9-prompt-engineering)
10. [State Management](#10-state-management)
11. [Error Handling Strategy](#11-error-handling-strategy)
12. [Security Architecture](#12-security-architecture)
13. [Performance & Scalability](#13-performance--scalability)
14. [Deployment Architecture](#14-deployment-architecture)
15. [Directory Structure](#15-directory-structure)
16. [Environment Variables](#16-environment-variables)
17. [Future Architecture Considerations](#17-future-architecture-considerations)

---

## 1. Architecture Overview

TraceNova follows a **three-tier architecture**: a React-based SPA on the frontend, a Node.js/Convex serverless backend, and the Anthropic Claude API as the AI reasoning engine. The system is stateless by design — no user data or code is persisted beyond the active browser session.

### Core Design Principles

- **Stateless & ephemeral** — Student code is never stored. Each session is isolated.
- **Streaming-first** — LLM responses are streamed token-by-token for low perceived latency.
- **Structured AI output** — The LLM is prompted to return strict JSON, parsed server-side before delivery to the client.
- **Progressive disclosure** — The UI renders each response section (explanation → steps → fixed code) as it arrives.
- **Separation of concerns** — Prompt construction, API communication, and response parsing are distinct backend modules.

### Technology Choices at a Glance

| Concern | Choice | Rationale |
|---|---|---|
| UI Framework | React + TypeScript | Component model maps cleanly to debug output sections |
| Bundler | Vite | Fast HMR for development; optimized production builds |
| Styling | Tailwind CSS | Utility-first; keeps design consistent without a heavy component library |
| Code Editor | Monaco Editor | Same engine as VS Code; syntax highlighting for Python, Java, C |
| Backend | Node.js (Express) or Convex | Express for simplicity; Convex for real-time / serverless at scale |
| AI API | Anthropic Claude (`claude-sonnet-4-6`) | Strong instruction-following; ideal for structured JSON output |
| File Parsing | Multer (Node) | Lightweight multipart form data handling |
| Hosting | Vercel (frontend) + Render/Convex (backend) | Zero-config deploys; scales automatically |

---

## 2. High-Level System Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          BROWSER (Client)                           │
│                                                                     │
│  ┌──────────────┐   ┌────────────────┐   ┌──────────────────────┐  │
│  │  Code Editor │   │ Language + ELI │   │   Output Panel       │  │
│  │  (Monaco)    │   │ 15 Controls    │   │  ┌────────────────┐  │  │
│  │              │   │                │   │  │ Error Explain  │  │  │
│  │  .py/.java/  │   │  [Python▼]     │   │  │ Debug Steps    │  │  │
│  │  .c upload   │   │  [ELI15 🧠]    │   │  │ Fixed Code     │  │  │
│  └──────┬───────┘   └───────┬────────┘   │  │ Heads Up ⚠️   │  │  │
│         │                   │            │  └────────────────┘  │  │
│         └──────────┬────────┘            └──────────────────────┘  │
│                    │                                                │
│              [Debug My Code 🚀]                                     │
└────────────────────┼────────────────────────────────────────────────┘
                     │  HTTPS POST /api/debug  (JSON + optional file)
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       BACKEND (Node / Convex)                       │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐ │
│  │  Rate Limiter│  │ File Parser  │  │     Prompt Builder        │ │
│  │  (IP-based)  │→ │ (Multer)     │→ │  system_prompt()          │ │
│  └──────────────┘  └──────────────┘  │  user_prompt(code, lang,  │ │
│                                      │    error, eli15)           │ │
│                                      └────────────┬──────────────┘ │
│                                                   │                │
│                                      ┌────────────▼──────────────┐ │
│                                      │    Claude API Client      │ │
│                                      │  (streaming, JSON mode)   │ │
│                                      └────────────┬──────────────┘ │
│                                                   │                │
│                                      ┌────────────▼──────────────┐ │
│                                      │    Response Parser        │ │
│                                      │  Validates JSON schema    │ │
│                                      │  Sanitizes code output    │ │
│                                      └────────────┬──────────────┘ │
└───────────────────────────────────────────────────┼────────────────┘
                                                    │  SSE stream / JSON
                                                    ▼
                                         Back to Browser → Renders
                                                    │
                     ┌──────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ANTHROPIC CLAUDE API                             │
│                  claude-sonnet-4-6                                  │
│              (External — HTTPS, authenticated)                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Frontend Architecture

### 3.1 Component Tree

```
<App>
├── <Toolbar>
│   ├── <Logo />
│   ├── <LanguageSelector />        # Python | Java | C pill toggle
│   └── <ELI15Toggle />             # Session-scoped boolean toggle
│
├── <WorkspaceLayout>               # Two-column on desktop, stacked on mobile
│   ├── <InputPanel>
│   │   ├── <FileUploadZone />      # Drag-drop or click; .py .java .c
│   │   ├── <CodeEditor />          # Monaco Editor instance
│   │   ├── <ErrorMessageInput />   # Optional textarea for error output
│   │   └── <DebugButton />         # Primary CTA; disabled during loading
│   │
│   └── <OutputPanel>
│       ├── <LoadingState />        # Shown during API call; friendly message
│       ├── <ErrorExplanation />    # Section 1 of response
│       ├── <DebugSteps />          # Numbered step list; Section 2
│       ├── <CorrectedCode />       # Monaco read-only + copy button; Section 3
│       ├── <BeginnerMistakes />    # "Heads Up" cards; Section 4
│       └── <LineByLineTable />     # Toggled on by "Explain My Code" button
│
└── <Footer />
```

### 3.2 Page States

| State | What the UI Shows |
|---|---|
| `idle` | Empty editor, placeholder text, disabled output panel |
| `loading` | Spinner + encouraging message ("Tracing your code…") |
| `streaming` | Output sections populate progressively as chunks arrive |
| `success` | Full output rendered; copy buttons active |
| `error` | Friendly error banner; "Try again" button |

### 3.3 Monaco Editor Configuration

```typescript
// src/components/CodeEditor/config.ts
export const monacoOptions = {
  language: 'python',           // dynamically updated on language toggle
  theme: 'vs-dark',
  fontSize: 14,
  lineNumbers: 'on',
  minimap: { enabled: false },
  scrollBeyondLastLine: false,
  wordWrap: 'on',
  automaticLayout: true,
  readOnly: false,
};
```

Language mapping:

```typescript
export const LANGUAGE_MAP: Record<SupportedLanguage, string> = {
  python: 'python',
  java:   'java',
  c:      'c',
};
```

---

## 4. Backend Architecture

### 4.1 Module Structure

```
server/
├── index.ts                  # Express app entry; registers routes + middleware
├── routes/
│   └── debug.ts              # POST /api/debug — main debug endpoint
│   └── explain.ts            # POST /api/explain — line-by-line mode
├── middleware/
│   ├── rateLimiter.ts        # IP-based rate limiting (10 req/hr per IP)
│   ├── fileParser.ts         # Multer config for .py .java .c uploads
│   └── inputValidator.ts     # Zod schema validation on incoming body
├── services/
│   ├── promptBuilder.ts      # Constructs system + user prompts
│   ├── claudeClient.ts       # Anthropic SDK wrapper; handles streaming
│   └── responseParser.ts     # Parses + validates structured JSON response
├── types/
│   └── debugTypes.ts         # TypeScript interfaces for all request/response shapes
└── utils/
    ├── sanitize.ts           # Strips dangerous content from code input
    └── languageDetect.ts     # Heuristic language detection from file extension
```

### 4.2 Express App Setup

```typescript
// server/index.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import { rateLimiter } from './middleware/rateLimiter';
import { debugRouter } from './routes/debug';
import { explainRouter } from './routes/explain';

const app = express();

app.use(helmet());
app.use(cors({ origin: process.env.FRONTEND_URL }));
app.use(express.json({ limit: '200kb' }));
app.use(rateLimiter);

app.use('/api/debug', debugRouter);
app.use('/api/explain', explainRouter);

app.listen(process.env.PORT || 3001);
```

### 4.3 Convex Alternative (Serverless)

If using Convex instead of Express, the architecture maps to:

```
convex/
├── debug.ts          # Convex action — calls Claude API (server-side)
├── explain.ts        # Convex action — line-by-line mode
└── schema.ts         # Schema (minimal; no persistent storage in v1.0)
```

Convex actions are preferred for v1.0 if real-time streaming and zero-infrastructure deployment are priorities.

---

## 5. AI / LLM Layer

### 5.1 Model Selection

| Model | Use Case | Why |
|---|---|---|
| `claude-sonnet-4-6` | All debug and explain requests | Best balance of speed, instruction following, and cost |

### 5.2 Claude API Call (with Streaming)

```typescript
// server/services/claudeClient.ts
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export async function streamDebugResponse(
  systemPrompt: string,
  userPrompt: string,
  onChunk: (chunk: string) => void
): Promise<void> {
  const stream = await client.messages.stream({
    model: 'claude-sonnet-4-6',
    max_tokens: 2048,
    system: systemPrompt,
    messages: [{ role: 'user', content: userPrompt }],
  });

  for await (const chunk of stream) {
    if (
      chunk.type === 'content_block_delta' &&
      chunk.delta.type === 'text_delta'
    ) {
      onChunk(chunk.delta.text);
    }
  }
}
```

### 5.3 Structured Output Schema

The LLM is instructed to return only valid JSON matching this schema:

```typescript
interface DebugResponse {
  error_explanation: string;
  error_type: string;                       // e.g. "SyntaxError", "RuntimeError"
  affected_lines: number[];
  debug_steps: {
    step: number;
    title: string;
    instruction: string;
    code_reference?: string;                // optional snippet being referenced
  }[];
  corrected_code: string;
  code_annotations: {
    line: number;
    note: string;
  }[];
  beginner_mistakes: {
    type: string;
    line: number;
    explanation: string;
    suggestion: string;
  }[];
  summary: string;                          // one-sentence summary of the fix
}

interface LineByLineResponse {
  line_explanations: {
    line_number: number;
    code: string;
    explanation: string;
  }[];
}
```

---

## 6. Data Flow & Sequence Diagrams

### 6.1 Standard Debug Request Flow

```
Student                  React Frontend            Node Backend           Claude API
   │                          │                         │                      │
   │── pastes code ──────────▶│                         │                      │
   │── (optional) error msg ─▶│                         │                      │
   │── clicks "Debug" ───────▶│                         │                      │
   │                          │── POST /api/debug ─────▶│                      │
   │                          │   { code, language,     │                      │
   │                          │     errorMsg, eli15 }   │                      │
   │                          │                         │── validate input     │
   │                          │                         │── build prompts      │
   │                          │                         │── POST /v1/messages ▶│
   │                          │                         │                      │── generates
   │                          │                         │◀── SSE stream ───────│   JSON
   │                          │◀── chunked response ────│                      │
   │◀── UI updates live ──────│                         │                      │
   │                          │                         │── parse + validate   │
   │                          │◀── final JSON ──────────│   JSON schema        │
   │◀── full output rendered ─│                         │                      │
```

### 6.2 File Upload Flow

```
Student                  React Frontend            Node Backend
   │                          │                         │
   │── drops .py file ───────▶│                         │
   │                          │── reads file as text    │
   │                          │── populates editor      │
   │                          │── detects language      │
   │── clicks "Debug" ───────▶│                         │
   │                          │── POST /api/debug ─────▶│
   │                          │   multipart/form-data   │── Multer parses file
   │                          │   + file blob           │── extracts code text
   │                          │                         │── continues as above
```

---

## 7. File Upload Pipeline

### 7.1 Multer Configuration

```typescript
// server/middleware/fileParser.ts
import multer from 'multer';
import path from 'path';

const ALLOWED_EXTENSIONS = ['.py', '.java', '.c'];
const MAX_FILE_SIZE = 100 * 1024; // 100 KB

const storage = multer.memoryStorage(); // no disk writes; ephemeral

const fileFilter = (_req: any, file: Express.Multer.File, cb: multer.FileFilterCallback) => {
  const ext = path.extname(file.originalname).toLowerCase();
  if (ALLOWED_EXTENSIONS.includes(ext)) {
    cb(null, true);
  } else {
    cb(new Error(`Unsupported file type: ${ext}. Upload .py, .java, or .c files.`));
  }
};

export const upload = multer({
  storage,
  limits: { fileSize: MAX_FILE_SIZE },
  fileFilter,
});
```

### 7.2 Language Detection from Extension

```typescript
// server/utils/languageDetect.ts
export function detectLanguage(filename: string): SupportedLanguage {
  const ext = filename.split('.').pop()?.toLowerCase();
  const map: Record<string, SupportedLanguage> = {
    py:   'python',
    java: 'java',
    c:    'c',
  };
  return map[ext ?? ''] ?? 'python';
}
```

---

## 8. API Contract

### POST `/api/debug`

**Request**

```json
{
  "code": "for i in range(10)\n  print(i)",
  "language": "python",
  "error_message": "SyntaxError: expected ':'",
  "eli15": false,
  "line_by_line": false
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `code` | string | ✅ | Raw source code (max 500 lines) |
| `language` | `"python" \| "java" \| "c"` | ✅ | Selected language |
| `error_message` | string | ❌ | Compiler/runtime error output |
| `eli15` | boolean | ❌ | Enable "Explain Like I'm 15" mode (default: false) |
| `line_by_line` | boolean | ❌ | Include line annotations (default: false) |

**Response (200)**

```json
{
  "error_explanation": "You forgot the colon at the end of your for loop...",
  "error_type": "SyntaxError",
  "affected_lines": [1],
  "debug_steps": [
    {
      "step": 1,
      "title": "Find the problem line",
      "instruction": "Look at line 1. In Python, a for loop must end with a colon (:).",
      "code_reference": "for i in range(10)"
    }
  ],
  "corrected_code": "for i in range(10):\n    print(i)",
  "code_annotations": [
    { "line": 1, "note": "Added missing ':' at end of for loop" }
  ],
  "beginner_mistakes": [
    {
      "type": "IndentationError",
      "line": 2,
      "explanation": "The print statement needs to be indented inside the loop.",
      "suggestion": "Add 4 spaces before print(i)"
    }
  ],
  "summary": "Added missing colon after for loop and fixed indentation."
}
```

**Error Responses**

| Status | Code | Meaning |
|---|---|---|
| 400 | `INVALID_INPUT` | Missing required fields or malformed JSON |
| 400 | `UNSUPPORTED_LANGUAGE` | Language not in allowed list |
| 413 | `CODE_TOO_LONG` | Code exceeds 500 lines |
| 415 | `UNSUPPORTED_FILE_TYPE` | Uploaded file extension not allowed |
| 429 | `RATE_LIMITED` | IP has exceeded 10 requests/hour |
| 500 | `LLM_ERROR` | Claude API returned an error or invalid JSON |
| 503 | `SERVICE_UNAVAILABLE` | Anthropic API unreachable |

---

## 9. Prompt Engineering

### 9.1 System Prompt

```
You are TraceNova, a patient and encouraging AI debugging assistant for beginner student programmers.

Your job is to help students understand their code errors and learn to fix them — not just hand them the answer.

RULES:
1. Always return valid JSON only. No markdown, no preamble, no explanation outside the JSON.
2. Use the exact JSON schema provided. Do not add or remove fields.
3. Keep all explanations kind, clear, and encouraging.
4. Never shame or criticize the student. Errors are normal.
5. When eli15 is true: use everyday analogies, short sentences, zero jargon. Explain as if talking to a curious 15-year-old.
6. When eli15 is false: you may use basic CS terms (loop, variable, function) but still avoid advanced terminology.
7. debug_steps must have between 3 and 8 steps.
8. corrected_code must be syntactically valid for the stated language.
9. Only flag beginner_mistakes you are confident about. Max 5. If none found, return an empty array.
10. Do not change anything in corrected_code beyond what is required to fix the reported error.
```

### 9.2 User Prompt Template

```typescript
// server/services/promptBuilder.ts
export function buildUserPrompt(params: DebugRequest): string {
  return `
Language: ${params.language}
ELI15 Mode: ${params.eli15 ? 'ON' : 'OFF'}

--- STUDENT CODE ---
${params.code}

--- ERROR MESSAGE ---
${params.error_message ?? 'No error message provided. Analyze the code for issues.'}

--- TASK ---
Debug the student's code. Return ONLY the JSON object matching the DebugResponse schema.
`.trim();
}
```

### 9.3 Prompt for Line-by-Line Mode

```typescript
export function buildLineByLinePrompt(code: string, language: string, eli15: boolean): string {
  return `
Language: ${language}
ELI15 Mode: ${eli15 ? 'ON' : 'OFF'}

--- STUDENT CODE ---
${code}

--- TASK ---
Explain every non-empty, non-comment line of this code in plain English.
Return ONLY the JSON object matching the LineByLineResponse schema.
Each explanation must be 1–2 sentences. ${eli15 ? 'Use simple language and analogies suitable for a 15-year-old.' : ''}
`.trim();
}
```

---

## 10. State Management

TraceNova uses **React local state + context** — no Redux or Zustand needed for v1.0.

### 10.1 Global Context

```typescript
// src/context/TraceNovaContext.tsx
interface TraceNovaState {
  code: string;
  language: SupportedLanguage;
  errorMessage: string;
  eli15: boolean;
  status: 'idle' | 'loading' | 'streaming' | 'success' | 'error';
  debugResult: DebugResponse | null;
  apiError: string | null;
}
```

### 10.2 State Transitions

```
idle
 │
 ├─ user submits → loading
 │
loading
 │
 ├─ first chunk received → streaming
 ├─ API error → error
 │
streaming
 │
 ├─ stream complete → success
 ├─ stream error → error
 │
success
 │
 └─ user edits code → idle

error
 │
 └─ user clicks retry → idle
```

---

## 11. Error Handling Strategy

### 11.1 Client-Side

- Network timeout: abort fetch after 30 seconds, show "This is taking longer than usual" message.
- JSON parse failure on response: show generic error, log to console.
- File too large: validate size before upload, show inline error next to upload zone.
- Empty code submission: disable the debug button; show hint text.

### 11.2 Server-Side

```typescript
// server/routes/debug.ts (error handling wrapper)
export async function handleDebugRequest(req: Request, res: Response) {
  try {
    const parsed = DebugRequestSchema.parse(req.body);   // Zod validation
    const systemPrompt = buildSystemPrompt(parsed.eli15);
    const userPrompt = buildUserPrompt(parsed);
    const rawResponse = await callClaude(systemPrompt, userPrompt);
    const result = parseAndValidateResponse(rawResponse);
    res.json(result);
  } catch (err) {
    if (err instanceof ZodError) {
      return res.status(400).json({ code: 'INVALID_INPUT', message: err.message });
    }
    if (err instanceof AnthropicAPIError) {
      return res.status(503).json({ code: 'LLM_ERROR', message: 'AI service unavailable' });
    }
    if (err instanceof JSONParseError) {
      return res.status(500).json({ code: 'LLM_ERROR', message: 'Unexpected AI response format' });
    }
    return res.status(500).json({ code: 'INTERNAL_ERROR', message: 'Something went wrong' });
  }
}
```

### 11.3 LLM Output Validation

Before sending the Claude response to the client, the server validates it against the Zod schema:

```typescript
// server/services/responseParser.ts
import { z } from 'zod';

const DebugResponseSchema = z.object({
  error_explanation: z.string().min(1),
  error_type: z.string(),
  affected_lines: z.array(z.number()),
  debug_steps: z.array(z.object({
    step: z.number(),
    title: z.string(),
    instruction: z.string(),
    code_reference: z.string().optional(),
  })).min(1).max(8),
  corrected_code: z.string(),
  code_annotations: z.array(z.object({ line: z.number(), note: z.string() })),
  beginner_mistakes: z.array(z.object({
    type: z.string(),
    line: z.number(),
    explanation: z.string(),
    suggestion: z.string(),
  })).max(5),
  summary: z.string(),
});

export function parseAndValidateResponse(raw: string): DebugResponse {
  const cleaned = raw.replace(/```json|```/g, '').trim();
  const parsed = JSON.parse(cleaned);
  return DebugResponseSchema.parse(parsed);
}
```

---

## 12. Security Architecture

### 12.1 Input Sanitization

All student code is treated as **untrusted text**. It is never executed server-side. Before forwarding to the LLM:

- Strip null bytes and control characters.
- Truncate to 500 lines / 50,000 characters maximum.
- Escape any content that could constitute prompt injection (e.g., lines beginning with `System:`, `IGNORE ALL PREVIOUS INSTRUCTIONS`).

```typescript
// server/utils/sanitize.ts
export function sanitizeCode(raw: string): string {
  return raw
    .replace(/\x00/g, '')                            // null bytes
    .replace(/\r\n/g, '\n')                          // normalize line endings
    .split('\n').slice(0, 500).join('\n')             // max 500 lines
    .substring(0, 50_000);                           // max 50K chars
}
```

### 12.2 Rate Limiting

```typescript
// server/middleware/rateLimiter.ts
import rateLimit from 'express-rate-limit';

export const rateLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,  // 1 hour window
  max: 10,                    // 10 requests per IP per hour
  standardHeaders: true,
  legacyHeaders: false,
  message: { code: 'RATE_LIMITED', message: 'Too many requests. Try again in an hour.' },
});
```

### 12.3 API Key Security

- `ANTHROPIC_API_KEY` is stored as a server-side environment variable only.
- Never exposed to the client or included in frontend bundles.
- The backend acts as a proxy — the browser never calls the Claude API directly.

### 12.4 CORS Policy

```typescript
cors({
  origin: process.env.FRONTEND_URL,   // e.g. https://tracenova.vercel.app
  methods: ['POST'],
  allowedHeaders: ['Content-Type'],
})
```

### 12.5 No Data Persistence

- No database in v1.0. No student code, no sessions, no logs with PII.
- Files uploaded via Multer are stored in memory only (`memoryStorage()`). They are garbage-collected at the end of the request lifecycle.

---

## 13. Performance & Scalability

### 13.1 Streaming Response Strategy

Instead of waiting for the full LLM response before rendering, TraceNova streams tokens from the Claude API and forwards them to the browser via **Server-Sent Events (SSE)**:

```
Claude API → SSE stream → Node backend → SSE stream → React frontend → Progressive render
```

This makes the interface feel fast even when total generation takes 5–8 seconds.

### 13.2 Frontend Performance

- Code editor is lazy-loaded (Monaco is large; ~2 MB).
- Output sections are rendered as individual React components; only updated sections re-render.
- Tailwind's JIT mode eliminates unused CSS.

### 13.3 Scalability Targets (v1.0)

| Metric | Target |
|---|---|
| Concurrent users | 500 |
| P95 response latency | < 8 seconds |
| Backend instances | 1–3 (auto-scale on Render) |
| Claude API concurrency | Governed by Anthropic rate limits |

### 13.4 Caching

No response caching in v1.0. Every debug request is unique. Caching is deferred until a clear repeating pattern (e.g., identical error + code submissions) emerges from analytics.

---

## 14. Deployment Architecture

```
┌──────────────────────────────────────────────────────────┐
│                       PRODUCTION                         │
│                                                          │
│  ┌─────────────────────┐     ┌────────────────────────┐  │
│  │   Vercel (Frontend) │     │  Render (Backend API)  │  │
│  │                     │     │                        │  │
│  │  React SPA          │────▶│  Node.js / Express     │  │
│  │  Static files + CDN │     │  POST /api/debug       │  │
│  │  Auto HTTPS         │     │  POST /api/explain     │  │
│  └─────────────────────┘     └───────────┬────────────┘  │
│                                          │               │
│                              ┌───────────▼────────────┐  │
│                              │  Anthropic Claude API  │  │
│                              │  (External)            │  │
│                              └────────────────────────┘  │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                       CI / CD                            │
│                                                          │
│  GitHub → push to main                                   │
│       ├── Vercel: auto-deploy frontend                   │
│       └── Render: auto-deploy backend                    │
└──────────────────────────────────────────────────────────┘
```

### 14.1 Environments

| Environment | Frontend URL | Backend URL | Notes |
|---|---|---|---|
| Development | `localhost:5173` | `localhost:3001` | Vite dev server + ts-node |
| Staging | `tracenova-staging.vercel.app` | `tracenova-api-staging.render.com` | Preview deployments on PRs |
| Production | `tracenova.app` | `api.tracenova.app` | Main branch deploys only |

---

## 15. Directory Structure

```
tracenova/
├── client/                          # React frontend
│   ├── public/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Toolbar/
│   │   │   │   ├── LanguageSelector.tsx
│   │   │   │   └── ELI15Toggle.tsx
│   │   │   ├── InputPanel/
│   │   │   │   ├── CodeEditor.tsx
│   │   │   │   ├── ErrorMessageInput.tsx
│   │   │   │   ├── FileUploadZone.tsx
│   │   │   │   └── DebugButton.tsx
│   │   │   └── OutputPanel/
│   │   │       ├── ErrorExplanation.tsx
│   │   │       ├── DebugSteps.tsx
│   │   │       ├── CorrectedCode.tsx
│   │   │       ├── BeginnerMistakes.tsx
│   │   │       └── LineByLineTable.tsx
│   │   ├── context/
│   │   │   └── TraceNovaContext.tsx
│   │   ├── hooks/
│   │   │   ├── useDebug.ts          # Manages API call + streaming state
│   │   │   └── useFileUpload.ts     # File drag-drop + parsing logic
│   │   ├── types/
│   │   │   └── debugTypes.ts
│   │   ├── utils/
│   │   │   └── formatCode.ts
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── index.html
│   ├── vite.config.ts
│   └── tailwind.config.ts
│
├── server/                          # Node.js backend
│   ├── src/
│   │   ├── index.ts
│   │   ├── routes/
│   │   │   ├── debug.ts
│   │   │   └── explain.ts
│   │   ├── middleware/
│   │   │   ├── rateLimiter.ts
│   │   │   ├── fileParser.ts
│   │   │   └── inputValidator.ts
│   │   ├── services/
│   │   │   ├── promptBuilder.ts
│   │   │   ├── claudeClient.ts
│   │   │   └── responseParser.ts
│   │   ├── types/
│   │   │   └── debugTypes.ts
│   │   └── utils/
│   │       ├── sanitize.ts
│   │       └── languageDetect.ts
│   ├── package.json
│   └── tsconfig.json
│
├── .env.example
├── .gitignore
├── docker-compose.yml               # Optional: local dev with Docker
└── README.md
```

---

## 16. Environment Variables

### Client (`client/.env`)

```bash
VITE_API_URL=http://localhost:3001    # Backend API base URL
```

### Server (`server/.env`)

```bash
ANTHROPIC_API_KEY=sk-ant-...          # Required: Anthropic API key
FRONTEND_URL=http://localhost:5173    # CORS allowed origin
PORT=3001                             # Server port (default 3001)
NODE_ENV=development                  # development | production
RATE_LIMIT_MAX=10                     # Max requests per IP per hour
RATE_LIMIT_WINDOW_MS=3600000          # 1 hour in ms
```

---

## 17. Future Architecture Considerations

These are explicitly out of scope for v1.0 but should be accounted for in architectural decisions now to avoid major refactors later.

### Persistent Sessions & History (v2.0)

If user accounts and session history are added, introduce a lightweight database (PlanetScale or Supabase) with a `sessions` table. Code and results would be stored encrypted at rest, tied to a user ID.

### Code Execution Sandbox (v2.0)

Running student code server-side to reproduce the error requires a sandboxed execution environment. Recommended approach: containerized execution using Docker + a strict resource limit policy (CPU/memory/time caps). Never execute on the main server process.

### Multi-Language Expansion (v2.0)

Add JavaScript, C++, and Rust. The architecture supports this via the `SupportedLanguage` union type and `LANGUAGE_MAP`. The main work is updating Monaco config and prompt tuning per language.

### IDE Extensions (v3.0)

VS Code and IntelliJ plugins would communicate directly with the backend API. The existing REST API contract is already suitable — no changes needed to `/api/debug`.

### Analytics & Feedback Loop (v2.0)

Add a lightweight event tracking layer (PostHog or Plausible) to measure which error types are most common, which debug steps students find confusing, and session completion rates. No PII; aggregate only.

---

*TraceNova Architecture v1.0 — Prepared by the TraceNova Engineering Team*  
*Refer to PRD.md for product requirements. For questions, open an issue in the project repository.*
