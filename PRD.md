# TraceNova — Product Requirements Document

**Version:** 1.0  
**Status:** Draft  
**Author:** Product Team  
**Last Updated:** March 2026

---

## 1. Executive Summary

**TraceNova** is an AI-powered code debugging assistant designed specifically for beginner and intermediate programmers. It bridges the gap between cryptic compiler errors and genuine understanding by translating technical error messages into plain language, walking users through step-by-step debugging logic, and surfacing corrected code — all inside a clean, student-friendly interface.

TraceNova targets one of the most underserved moments in learning to code: the moment a student is stuck, confused, and about to give up.

---

## 2. Problem Statement

Learning to program is hard. Error messages are often terse, jargon-heavy, and assume prior knowledge that beginners simply don't have. Existing tools like Stack Overflow assume you already understand the domain vocabulary. Generalist AI chatbots help, but require the student to articulate the problem clearly — a skill that itself takes years to develop.

Beginner programmers face a recurring cycle:

1. Write code
2. Hit an error
3. Copy the error into Google
4. Read an answer they don't fully understand
5. Patch the code blindly
6. Repeat

TraceNova breaks this cycle by making the debugging process itself educational — not just fixing code, but teaching students *why* something went wrong and *how* to think about fixing it.

---

## 3. Goals and Success Metrics

### Primary Goals

- Help students understand errors, not just fix them
- Reduce time-to-understanding for beginner programmers
- Build debugging intuition through repeated, guided exposure
- Be genuinely usable for ages 13 and up

### Success Metrics

| Metric | Target (6-month) |
|---|---|
| Monthly Active Users | 10,000+ |
| Session error-to-understanding rate | ≥ 80% user-reported |
| Average session length | 4–8 minutes |
| Returning users (week-over-week) | ≥ 40% |
| "Explain Like I'm 15" feature usage | ≥ 35% of sessions |

---

## 4. Target Users

### Primary: Beginner Students (Ages 13–22)
High school and university students learning C, Python, or Java for the first time. They encounter errors daily and often lack the vocabulary to search for solutions effectively.

### Secondary: Self-Taught Developers
Adults learning to code via online courses or bootcamps. They have motivation but limited access to mentors or teaching assistants.

### Tertiary: Educators
Teachers and TAs who want to direct struggling students to a reliable, patient first-line debugging resource — freeing up classroom time for deeper instruction.

---

## 5. Supported Languages (v1.0)

- Python (`.py`)
- C (`.c`)
- Java (`.java`)

These three languages cover the majority of introductory programming curricula worldwide and represent the most common sources of beginner frustration.

---

## 6. Core Features

### 6.1 Error Explanation in Plain Language

When a student submits code with an error, TraceNova produces a human-readable explanation that avoids jargon. Instead of "NullPointerException at line 14," the student reads something like: *"You're trying to use a variable that hasn't been given a value yet. Think of it like trying to open a box that was never filled."*

**Requirements:**
- Error message must be identified and isolated before explanation
- Explanation must be written at an 8th–10th grade reading level by default
- No assumption of prior CS vocabulary in the explanation text
- Explanation must identify the exact line(s) involved

### 6.2 Step-by-Step Debugging Walkthrough

TraceNova doesn't just name the problem — it walks the student through a debugging sequence, teaching the *process* of finding and fixing bugs rather than just handing over a solution.

**Requirements:**
- Walkthrough presented as numbered steps (3–6 steps typical)
- Each step explains what to look at and why
- Steps progress from observation → hypothesis → fix
- Student can expand or collapse individual steps
- Steps must reference specific line numbers in the submitted code

### 6.3 Corrected Code Suggestion

After the walkthrough, TraceNova shows a corrected version of the code with the fix applied.

**Requirements:**
- Corrected code displayed in a syntax-highlighted code block
- Diff view available: shows original vs. corrected side-by-side
- Changes annotated with inline comments explaining each modification
- One-click copy to clipboard
- "What changed?" summary shown above the code block

### 6.4 Common Beginner Mistake Detection

TraceNova proactively identifies patterns that indicate beginner-specific mistakes — not just syntax errors, but logic errors, style issues, and structural anti-patterns common at the learning stage.

**Detected patterns include (non-exhaustive):**
- Off-by-one errors in loops
- Uninitialized variables
- Missing return statements
- Incorrect indentation (Python)
- Confusion between `=` and `==`
- Unused imports or variables
- Infinite loop conditions
- Type mismatch in function arguments
- Array out-of-bounds access
- Forgetting to close file handles

**Requirements:**
- Detected patterns flagged with a yellow "Heads Up" badge
- Each flag includes a one-sentence explanation of why it matters
- Patterns ranked by severity (error vs. warning vs. style note)

### 6.5 File Upload Support

Students can upload their source files directly rather than pasting code manually.

**Requirements:**
- Supported file types: `.py`, `.c`, `.java`
- Max file size: 500KB
- File contents displayed in editor after upload with syntax highlighting
- Upload via drag-and-drop or file picker
- Pasting code directly also fully supported (no account required)

### 6.6 Line-by-Line Code Explanation

For students who want to understand what their code does — not just what's wrong with it — TraceNova offers a line-by-line explanation mode.

**Requirements:**
- Triggered by clicking "Explain My Code" button
- Each line or logical block annotated with a plain-language tooltip or sidebar note
- Grouping of related lines (e.g., a loop body explained as a unit, not line-by-line)
- Annotations collapsed by default, expandable on click
- Available for both erroneous and correct code

### 6.7 "Explain Like I'm 15" Mode

A toggle that rewrites all AI-generated output in maximally accessible language: shorter sentences, simpler vocabulary, relatable analogies, zero jargon.

**Requirements:**
- Toggle accessible from top navigation bar and from any explanation block
- When enabled, all subsequent outputs (explanations, walkthrough, annotations) use simplified language
- Analogies drawn from everyday life (e.g., variables as labeled boxes, loops as repeating a task until done)
- Mode persists for the session; can be toggled off at any time
- Visual indicator (badge or icon change) confirms mode is active

---

## 7. Technical Architecture

### 7.1 Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React (TypeScript) |
| Backend / BaaS | Convex (real-time database + serverless functions) |
| AI / LLM | Anthropic Claude API (claude-sonnet-4-6) |
| Code Editor | Monaco Editor (VS Code engine) |
| Auth | Clerk or Convex Auth |
| File Storage | Convex file storage |
| Hosting | Vercel (frontend) + Convex cloud (backend) |

### 7.2 Data Flow

```
User submits code (paste or upload)
        ↓
Frontend sends code + language + mode flag to Convex function
        ↓
Convex function calls Claude API with structured prompt
        ↓
Claude returns: error analysis, walkthrough steps, corrected code, detected patterns
        ↓
Response parsed and stored in Convex DB (session-scoped)
        ↓
Frontend renders structured output reactively
```

### 7.3 LLM Prompt Design

Each API call to Claude will use a structured system prompt that instructs the model to respond as a patient, encouraging programming tutor. The prompt enforces:

- Output in structured JSON (error explanation, steps array, corrected code, pattern flags)
- Reading level calibration based on active mode (default vs. ELI15)
- Language-specific context (C pointer errors differ from Python indentation errors)
- Tone: warm, non-judgmental, encouraging

### 7.4 Security and Privacy

- No code submitted by users is stored beyond the session unless the user explicitly saves it
- No account required for core functionality (guest sessions)
- Uploaded files are scanned for size and MIME type before processing
- No user code is used for model training

---

## 8. User Interface Requirements

### 8.1 Layout

The main interface consists of three panels:

**Left Panel — Code Input**
- Monaco-based code editor with syntax highlighting
- Language selector (Python / C / Java)
- Upload button + drag-and-drop zone
- "Debug My Code" primary CTA button

**Center Panel — Analysis Output**
- Error explanation card (always first)
- Step-by-step debugging accordion
- Detected patterns list (if any)
- "Explain My Code" secondary action

**Right Panel — Corrected Code**
- Syntax-highlighted corrected code block
- Diff toggle (original vs. corrected)
- Copy button
- "What changed?" summary

On mobile, panels stack vertically. Center panel is prioritized in the viewport.

### 8.2 Visual Design Principles

- Clean, distraction-free interface
- Color-coded output: red for errors, yellow for warnings, green for corrected code
- Friendly iconography (no intimidating terminal aesthetics)
- Dark mode and light mode support
- Accessible: WCAG 2.1 AA compliance target

---

## 9. Out of Scope for v1.0

The following are explicitly excluded from the initial release:

- Code execution / runtime environment (no running code in the browser)
- Support for languages beyond Python, C, and Java
- Collaborative / shared sessions
- Teacher dashboard or LMS integration
- Mobile native app (web-responsive only)
- Plagiarism detection
- Version history or code diff tracking across sessions

---

## 10. Phased Roadmap

### Phase 1 — MVP (Months 1–3)
Core debugging flow: paste code, get explanation, get fix. Python only. No auth required. Basic ELI15 mode.

### Phase 2 — Expansion (Months 4–6)
Add C and Java support. File upload. Line-by-line explanation. Beginner pattern detection. Light/dark mode. User accounts with session history.

### Phase 3 — Growth (Months 7–12)
Teacher dashboard. Embeddable widget for coding bootcamps. Analytics on most common errors by language. API for third-party integrations. Mobile-optimized PWA.

---

## 11. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| LLM produces incorrect fix | Medium | High | Show disclaimer; encourage students to understand the fix before applying it |
| Over-reliance on tool reduces learning | Medium | Medium | Reinforce "understanding" over "copying"; require student to articulate what changed |
| API costs scale poorly with usage | Medium | Medium | Implement per-session token budgets; cache common error patterns |
| Students submit malicious code | Low | Low | No code execution on server; LLM input is read-only analysis |
| Scope creep delays MVP | High | High | Strict v1.0 freeze on feature list; weekly milestone reviews |

---

## 12. Glossary

| Term | Definition |
|---|---|
| ELI15 Mode | "Explain Like I'm 15" — simplified language mode for maximum accessibility |
| Walkthrough | The step-by-step debugging sequence generated per submission |
| Pattern Flag | A detected beginner-specific mistake surfaced proactively, even if not the primary error |
| Session | A single browser session; code and results persist until tab is closed or cleared |
| Diff View | Side-by-side comparison of original and corrected code |

---

*TraceNova — Making debugging make sense.*
