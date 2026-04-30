# CLAUDE.md — Progress Journal

This file tells Claude everything it needs to know to work effectively on this project across sessions.

---

## Project Overview

**Progress Journal** is a personal, mobile-first web app for recording daily completed tasks and reflections. It is a private tool used exclusively by Tim Nash (tim@nash.je).

The app has three main screens:
- **Write** — log tasks for today (by list or by time), with voice recording and screenshot/image import
- **Read** — browse and review past entries
- **Review** — AI-powered summaries across date ranges and task lists

Development tasks related to this project are tagged **[Zipf]** in the journal itself.

---

## Architecture

| Layer | Technology | Notes |
|---|---|---|
| Frontend UI | `index.html` on GitHub Pages | Single self-contained file |
| Backend API | Google Apps Script (`Code.gs`) | Deployed as a web app |
| Database | Notion | Via Notion API |
| AI | Anthropic Claude API | OCR, task extraction, summaries |
| Auth | Google OAuth + session tokens | Single allowed email: tim@nash.je |

### Data flow
```
Browser (index.html)
  → Google Apps Script (Code.gs)
    → Notion API (read/write entries)
    → Anthropic API (AI features)
```

### Notion data model
Each entry is a Notion page with:
- `Name` — date string (e.g. "April 8, 2026")
- `Date` — date property
- `Milestones` — float
- `Reflection` — rich text (may be plain string or JSON with `general`, `sections`, `dayStart`)
- `Tasks` — rich text, one task per line, format: `• [List] [HH:MM] [Nm] [★] Task title`

### Task format in Notion
```
• [List] Task title
• [List] [HH:MM] Task title
• [List] [HH:MM] [Nm] Task title
• [List] [HH:MM] [Nm] ★ Task title
```
- List names: Work, Personal, Basic, Zipf, Other, Tasks, etc.
- Time and duration are optional
- ★ = starred/significant task

---

## GitHub Workflow

### Repository
- **Repo**: `https://github.com/timfnash/progress_journal`
- **Live site**: `https://timfnash.github.io/progress_journal/`
- Tim will provide a personal access token (PAT) at the point of committing.
- The PAT should be used for HTTPS authentication: `https://<PAT>@github.com/timfnash/progress_journal.git`

### Commit process (always follow this order)
1. Make the code changes using `str_replace` for targeted edits, or rewrite the full file if the change is large/structural.
2. **Show Tim a clear summary of what changed** — bullet points describing each modification.
3. **Ask for confirmation** before committing ("Ready to commit?").
4. Only commit after Tim explicitly confirms.

### Commit message format
Use screen/feature tags in square brackets:

```
[Write] Fix drag handle alignment on task list
[Read] Show times in lowercase h and m
[Review] Remove bullets from AI summary output
[Auth] Improve session handling on mobile
[General] Refactor task parsing utility
```

Multiple changes in one commit: use the most prominent tag and list others in the body.

---

## Coding Conventions

### File structure
- **Single file only**: everything lives in `index.html` — HTML, CSS, and JavaScript.
- Do not split into separate `.js` or `.css` files.
- No build tools, no bundlers, no npm.

### CSS
- Vanilla CSS only — no Tailwind, no CSS frameworks.
- Use CSS custom properties (variables) for colours and spacing where it makes the code cleaner.
- Do not change the existing colour theme without being asked.
- Mobile-first: design for a narrow viewport (~375px), then enhance for wider screens.

### JavaScript
- Vanilla JS only — no frameworks, no libraries unless already present in the file.
- Prefer `async/await` over `.then()` chains.
- Keep functions small and named clearly.
- No TypeScript.

### Mobile / iOS Safari
- Primary target: **iOS Safari on iPhone**.
- Always consider: touch targets (min 44px), safe area insets, scroll behaviour, keyboard appearance.
- Avoid hover-only interactions.
- Test mentally for: tap vs scroll conflicts, fixed position elements with keyboard open, input zoom (font-size ≥ 16px to prevent auto-zoom).
- Drag-and-drop: require tap-and-hold to initiate (avoid accidental drags during scroll).

### Editing strategy
- For **small/targeted changes**: use `str_replace` to edit only the relevant section. Do not rewrite the whole file.
- For **large structural changes**: rewrite the full file, but flag this clearly to Tim before proceeding.
- Always read the current file content before making edits to avoid stale context.

---

## Key Features & Behaviour

### Write screen — By List view
- Tasks grouped by list name
- Drag-and-drop reordering (tap-and-hold to initiate)
- Star (★) toggle per task
- Edit task title inline
- Delete task
- Add new task to a list
- Import tasks from: screenshot (via Claude vision), voice recording (transcription), or manual entry

### Write screen — By Time view
- Tasks displayed as time blocks on a timeline
- Blocks should not overlap — show side by side in separate columns if they do
- Dragging a block moves its time (tap-and-hold required)
- "Day start" marker at the 5-minute slot before the first task

### Read screen
- Browse past entries by date
- Shows tasks and reflection for each day

### Review screen
- Select a list and date range
- AI generates a punchy progress summary (see prompt style below)
- Drilldown into individual list entries

### AI summary style (for `generateListSummary`)
- Punchy, direct — not LinkedIn-style cheerleading
- Bold key achievements with `**markdown**`
- No bullet characters — each item on its own line with a blank line between
- 4–8 items max
- No mention of time, duration, or hours spent
- Weave in feelings/context from reflections if present

---

## Reflection field format

The `Reflection` field in Notion stores either:
- A plain string (older entries)
- A JSON object (newer entries):
```json
{
  "general": "Free text reflection",
  "dayStart": "08:15",
  "sections": [
    { "list": "Zipf", "text": "List-specific reflection text" }
  ]
}
```

---

## Authentication

- Only `tim@nash.je` is authorised.
- Auth uses Google OAuth ID tokens on first sign-in, then session tokens (stored in Google Apps Script properties) for subsequent requests.
- Sessions expire after 30 days and are bound to User-Agent.
- Multiple sessions are allowed (phone + desktop).
- Session actions: `createSession`, `revokeSession`, `revokeAllSessions`.

---

## Google Apps Script API

Base URL: stored in `index.html` as `SCRIPT_URL` constant.

All requests are POST with JSON body:
```json
{
  "action": "actionName",
  "sessionToken": "...",
  "userAgent": "...",
  "params": { }
}
```

### Available actions
| Action | Params | Description |
|---|---|---|
| `getEntryForDate` | `date` (YYYY-MM-DD) | Fetch single entry |
| `saveEntry` | `payload` | Create new entry |
| `updateEntry` | `pageId`, `payload` | Update existing entry |
| `getEntries` | `startDate`, `endDate` | Fetch range of entries |
| `ocrImage` | `base64Data`, `mediaType` | Transcribe handwritten image |
| `extractTasksFromImage` | `base64Data`, `mediaType` | Extract tasks from To Do screenshot |
| `extractTasksFromTranscript` | `transcript` | Extract tasks from voice transcript |
| `generateListSummary` | `list`, `tasks`, `reflections`, `dateRange` | AI summary |
| `renameTask` | `list`, `oldTitle`, `newTitle` | Rename task across all entries |
| `createSession` | — | Create session token |
| `revokeSession` | — | Sign out this device |
| `revokeAllSessions` | — | Sign out everywhere |

---

## Things Claude Should Never Do

- Do not change the colour theme unless explicitly asked.
- Do not split `index.html` into multiple files.
- Do not introduce CSS frameworks (Tailwind, Bootstrap, etc.).
- Do not introduce JavaScript frameworks (React, Vue, etc.).
- Do not commit to GitHub without Tim's explicit confirmation.
- Do not expose or log API keys in generated code.
- Do not add features that weren't requested — keep changes focused.

---

## Session Startup Checklist

At the start of each working session:
1. Ask Tim for his PAT if commits will be needed (he shares it at commit time, not upfront).
2. Read the current `index.html` from the repo via `https://raw.githubusercontent.com/timfnash/progress_journal/main/index.html` to avoid working from stale context.
3. Confirm the task(s) to work on before writing any code.
