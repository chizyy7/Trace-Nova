# TraceNova Code Prompt

TraceNova – Code Generation Prompt

Project Overview

Build TraceNova, an AI-powered debugging assistant designed for beginner programmers.
The system translates confusing compiler errors into clear explanations, provides a step-by-step
debugging walkthrough,
and returns corrected code with annotations. The goal is to help students understand why their code
failed, not just fix it.

TraceNova supports Python, C, and Java and targets beginner students learning programming for the
first time.
The interface should be clean, friendly, and educational.

Core Product Features

TraceNova must implement the following capabilities:

(cid:127) Error explanation in plain language.
(cid:127) Step-by-step debugging walkthrough.
(cid:127) Corrected code output with annotations.
(cid:127) Beginner mistake detection.
(cid:127) File upload support (.py, .java, .c).
(cid:127) Line-by-line code explanation mode.
(cid:127) "Explain Like I'm 15" (ELI15) mode using simple language and analogies.

System Architecture

TraceNova uses a three-tier architecture:

Frontend: React + TypeScript SPA with Monaco Editor.
Backend: Node.js server or Convex serverless functions handling API requests, prompts, and parsing.
AI Layer: Anthropic Claude API (claude-sonnet-4-6) for debugging analysis.

The system is stateless and ephemeral — student code is not permanently stored.

Tech Stack

Frontend:
React
TypeScript
Tailwind CSS
Monaco Editor
Vite

Backend:
Node.js with Express or Convex
Multer for file uploads
Zod for input validation

AI:
Anthropic Claude API (claude-sonnet-4-6)

Deployment:
Frontend on Vercel
Backend on Render or Convex cloud.

Main API Endpoint

POST /api/debug

Request:
{
  "code": "...source code...",
  "language": "python | java | c",
  "error_message": "...optional compiler error...",
  "eli15": false,
  "line_by_line": false
}

Response:
{
  error_explanation: string,
  error_type: string,
  affected_lines: number[],
  debug_steps: [{ step, title, instruction }],
  corrected_code: string,
  code_annotations: [{ line, note }],
  beginner_mistakes: [{ type, line, explanation, suggestion }],
  summary: string
}

The AI must return strict JSON matching this schema.

Frontend UI Layout

Three-panel layout:

Left: Code editor, language selector, file upload.
Center: Error explanation, debugging steps, mistake detection.
Right: Corrected code with syntax highlighting and diff view.

The interface must support dark mode, responsive layout, and beginner-friendly visuals.

Security Requirements

(cid:127) User code must never be executed on the server.
(cid:127) Uploaded files must be validated for type and size.
(cid:127) All inputs must be sanitized before sending to the AI.
(cid:127) API requests should be rate limited.
(cid:127) Student code must not be stored beyond the active session.

Development Goal

Build the complete full-stack TraceNova application including:

(cid:127) React frontend interface
(cid:127) Backend API and middleware
(cid:127) Prompt construction and LLM integration
(cid:127) Streaming AI responses
(cid:127) Structured JSON parsing
(cid:127) UI rendering of debugging results

The final system should allow students to submit code and quickly understand errors and fixes.

