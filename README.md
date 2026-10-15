# Browser Use Tool Harness

**Browser Use Tool Harness** is an interactive terminal application and agent engineering harness that enables language models—running in the cloud via **Azure AI Foundry**, on-device via **Foundry Local**, or in a **Hybrid** split—to operate a real web browser through [`playwright-cli`](https://github.com/microsoft/playwright-cli).

Describe a task in natural language (for example, *"summarize two AI articles from today and yesterday and save the summary as Markdown"*). The harness decomposes the request into a typed plan, drives a persistent Chromium session, verifies completion against concrete page evidence, pauses for inline human approval before irreversible actions, and learns reusable navigation guides over time.

## Demo

https://github.com/user-attachments/assets/dce0f639-42f7-4f53-9845-302ab0420610

> *Watch the harness decompose a natural-language task, drive Chromium through `playwright-cli`, verify completion evidence, and track token usage in the terminal UI ([direct MP4 file](assets/Browser-Tool-Use-Harness-Demo.mp4)).*

---

## Why a Harness?

Connecting a raw language model directly to browser automation tools quickly exposes recurring failure modes: context windows explode with full DOM/accessibility snapshots, models hallucinate completion based on old page state, multi-paragraph messages get truncated or sent early on `Enter`, and brittle RPA-style selector scripts break as soon as a layout changes.

This project explores the **systems engineering around the model** required to make browser agents reliable, token-efficient, and safe:

- **Independent Planner / Executor Roles (Cloud, Local, or Hybrid).** Pair a reasoning model for task planning with a fast tool-calling model for browser execution across **Azure AI Foundry** and **Foundry Local**, or run everything 100% locally on your own hardware.
- **Typed Task Intent & Evidence-Based Guardrails.** Every plan step carries an explicit type (`browser`, `research`, `write`, `send`, `verify`, `playbook`). Tasks cannot be closed on model claims alone—the harness verifies that required actions actually occurred in the current session.
- **Transactional One-Message Composition (`compose` + `verify-message`).** Long, multi-paragraph drafts (up to 40,000 characters) are typed in safe chunks using `Shift+Enter` into a single unsent draft, verified against the visible recipient header and SHA-256 digest before submission, and confirmed in the conversation view afterward.
- **Authoritative Civil Time & Dated Research Quotas.** Both planner and executor receive the host's local civil date and timezone. Requests like *"two articles from today and two from yesterday"* enforce separate per-date quotas verified against structured article metadata (`JSON-LD`, OpenGraph, `<article><time>`), never guessed from URLs.
- **Token-Efficient by Design.** Main-region text extraction (`read <url>`), multi-label element search (`find Connect | Conectar`), post-action DOM delta diffs, and prompt-cache-friendly batched context compaction keep token usage low and response times fast.
- **Inline Human-in-the-Loop (HITL).** Irreversible actions (purchases, deletions, unrequested sends) and ambiguous instructions surface inline approval and multiple-choice question panels directly inside the scrollable conversation timeline.
- **Generative Learning—Never Brittle RPA.** Demonstrations recorded with `/teach` are distilled into semantic landmark guides (visible labels, URLs, and user notes) rather than deterministic selector replay. Two-level session and long-term memory automatically measures wasted steps and compacts itself without losing known sites.

---

## Contents

1. [Demo](#demo)
2. [Why a Harness?](#why-a-harness)
3. [Architecture](#architecture)
4. [Requirements](#requirements)
5. [Quick Start](#quick-start)
6. [Model Configuration (`/models`, Cloud & Local Providers)](#model-configuration-models-cloud--local-providers)
7. [Terminal Interface & Controls](#terminal-interface--controls)
8. [How the Harness Works](#how-the-harness-works)
9. [Generative Learning: Memory & Playbooks](#generative-learning-memory--playbooks)
10. [Session Workspace, Logs & Usage Reports](#session-workspace-logs--usage-reports)
11. [Browser Session & Manual Sign-In](#browser-session--manual-sign-in)
12. [Security & Privacy](#security--privacy)
13. [Project Structure](#project-structure)
14. [Testing & Verification](#testing--verification)
15. [Public Sharing Checklist](#public-sharing-checklist)

---

## Architecture

```text
                 ┌────────────────────────────── Terminal UI (Textual) ──────────────────────────────┐
  User Request ─►│  Chat Timeline · Live Plan Panel · Inline Approvals & Questions · `/models` Modal │
                 └──────────────────┬────────────────────────────────────────────────────────────────┘
                                    │
                 ┌──────────────────▼───────────────────┐   Typed Plan (`browser`, `research`,
                 │ Planner (Optional: Azure or Local)   │   `write`, `send`, `verify`, `playbook`)
                 │ Decomposes request or asks question  │ ─────────────────────────────────────┐
                 └──────────────────────────────────────┘                                      │
                                                                                               ▼
                 ┌──────────────────────────────────────┐   Tool Calls    ┌──────────────────────────────────┐
                 │ Executor (Azure Foundry or Local)    │ ──────────────► │ Harness Gate & Tool Dispatcher   │
                 │ Loop: Think → Act → Observe → Verify │ ◄────────────── │ • Safety & recipient checks      │
                 └──────────────────┬───────────────────┘   Delta / Text  │ • Source date & quota validation │
                                    │                                     │ • Stuck-ladder escalation        │
                                    ▼                                     └─────────┬──────────────┬─────────┘
                 ┌──────────────────────────────────────┐                           │              │
                 │ Generative Learning & Persistence    │ ◄─────────────────────────┘              ▼
                 │ • `memory/` (session & long-term)    │                 ┌──────────────────────────────────┐
                 │ • `playbooks/` (`/teach` guides)     │                 │ `playwright-cli` → Chromium      │
                 │ • `workspaces/` (pages, shots, files)│                 │ Persistent local `.profile/`     │
                 └──────────────────────────────────────┘                 └──────────────────────────────────┘
```

| Layer | Responsibility |
|---|---|
| **Planner** (`planner.py`, `plan.py`) | Maps complex, multi-item, or consequential requests into up to 8 typed tasks (or asks a clarifying multiple-choice question). When disabled, the executor plans directly. |
| **Executor** (`agent.py`) | Runs the *think → act → observe* loop, manages context compaction, enforces completion evidence, and drives the stuck-recovery ladder. |
| **Provider Runtime** (`model_config.py`, `foundry.py`, `local_foundry.py`) | Manages credential-free configuration, Azure Entra ID bearer authentication, and Foundry Local CLI lifecycle (status check, loopback discovery, model loading, and tool-calling validation). |
| **Browser Tool** (`browser_tool.py`) | Enforces a strict command allowlist, executes `playwright-cli` without a shell, extracts main-region text and publication metadata, computes DOM change diffs, and handles transactional message composition. |
| **Learning Engine** (`memory.py`, `playbooks.py`) | Records per-turn telemetry and wasted steps, consolidates and self-reviews long-term navigation lessons, and turns `/teach` browser demonstrations into semantic landmark guides. |

---

## Requirements

- **Operating System:** Windows, macOS, or Linux *(native Win32 clipboard helpers for right-click file/text copy-paste and Windows Terminal `Shift+Enter` keybinding inspection are Windows-specific; standard shortcuts work everywhere)*.
- **Python:** 3.11 or later.
- **Node.js:** 18 or later (for the local `@playwright/cli` package).
- **Model Provider (choose one or both):**
  - **Azure AI Foundry:** An Azure AI Foundry / Azure OpenAI endpoint, model deployment(s), and Azure CLI sign-in (`az login`).
  - **Foundry Local:** The [Foundry Local CLI](https://learn.microsoft.com/azure/foundry-local/reference/reference-cli) (`foundry`) and a local chat model (the executor role requires a model with native tool-calling support, `supportsToolCalling: true`).

---

## Quick Start

The commands below use PowerShell on Windows (on macOS/Linux, substitute `source .venv/bin/activate` and `export VAR=value`). No account, endpoint, deployment name, or machine-specific path is hardcoded in the repository.

```powershell
git clone https://github.com/marcoszanre/browser-use-tool-harness.git
Set-Location browser-use-tool-harness

# 1. Create the Python virtual environment and install dependencies
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt

# 2. Install playwright-cli locally and download Playwright Chromium
npm ci
npx playwright-cli install-browser chromium

# 3. Launch the terminal UI
.\.venv\Scripts\python.exe browser_harness.py
```

Once inside the application, run `/models` (or click the **Models** button) to select a preset or configure your cloud/local endpoints before submitting your first task.

### Option A: Using Azure AI Foundry

Sign in with Azure CLI (`az login`). Authentication uses `DefaultAzureCredential` (`https://ai.azure.com/.default`)—no API keys or tokens are ever stored in files.

You can configure your endpoint and deployments interactively via `/models` or via environment variables:

```powershell
az login
$env:BROWSER_HARNESS_FOUNDRY_ENDPOINT = "https://<your-resource>.services.ai.azure.com/openai/v1/"
$env:BROWSER_HARNESS_EXECUTOR_MODEL = "gpt-4.1-mini"
$env:BROWSER_HARNESS_PLANNER_MODEL = "gpt-4.1"   # optional; set to "0" or "off" to disable
.\.venv\Scripts\python.exe browser_harness.py
```

### Option B: Using Foundry Local (100% On-Device)

Install Foundry Local and inspect or download a tool-calling model:

```powershell
foundry model list
foundry model info qwen2.5-1.5b --output json
foundry model download qwen2.5-1.5b
```

Then open `/models` in the TUI, click **Foundry Local (GPU)** (or select `Foundry Local` as the provider and enter your local model alias), and click **Save and check**. The harness automatically:
1. Checks `foundry server status` and starts the local server if needed.
2. Discovers the active HTTP loopback endpoint (`127.0.0.1` / `localhost`).
3. Verifies that the executor model supports native tool calling (`supportsToolCalling: true`).
4. Loads the model into memory and runs a synthetic diagnostic check.

---

## Model Configuration (`/models`, Cloud & Local Providers)

Running `/models` (or clicking **Models**) opens a centered modal dialog with one-click presets and independent role cards for **Executor (Browser Actions)** and **Planner (Task Breakdown)**:

- **Foundry Local (GPU):** Uses Foundry Local for both planning and execution (loopback port auto-discovered).
- **Foundry Normal (Azure):** Uses Azure AI Foundry for both planning (with low reasoning effort) and fast execution.
- **Hybrid (Azure + Local):** Uses an Azure AI Foundry reasoning model to build the task plan and a Foundry Local model to execute browser steps on-device.
- **Local Fast (No Plan):** Uses Foundry Local for execution with the separate planner disabled (the executor plans inline).

Clicking **Save and check** runs a live diagnostic before closing:
- For the **Executor**, it requires the model to emit a valid native `diagnostic_ping(ok=true)` tool call.
- For the **Planner**, it requires the model to return a valid JSON task plan that passes schema validation.
- Diagnostics never open or manipulate the browser, and a failed local check never silently falls back to the cloud.

### Persistent Settings (`settings.json`)

`settings.example.json` provides a clean, credential-free template. Saving in `/models` writes atomically to `settings.json` (which is ignored by Git):

| Field | Role / Scope | Description |
|---|---|---|
| `provider` | `executor`, `planner` | `"azure"` or `"local"` (`"off"` is also valid for `planner`) |
| `model` | `executor`, `planner` | Azure deployment name, Foundry Local alias, or full model ID |
| `endpoint` | `executor`, `planner` | Public Azure HTTPS endpoint (`*.services.ai.azure.com`, `*.openai.azure.com`, `*.cognitiveservices.azure.com`), or blank for Foundry Local auto-discovery |
| `reasoning` | `executor`, `planner` | Optional Azure reasoning effort (`""`, `"none"`, `"low"`, `"medium"`, `"high"`) |
| `max_tokens` | `executor`, `planner` | Maximum completion tokens (`1`–`32768`, default `4000`) |
| `planner_scope` | Top-level | `"actions"` (use planner for requests that act on accounts/people, lists, or date quotas) or `"all"` (every request) |
| `allow_local_downloads` | Top-level | `false` by default; when `true`, allows downloading uncached Foundry Local models |
| `azure_endpoint` | Top-level | Retains your Azure HTTPS endpoint when switching between local and cloud presets |

### Environment Variables

Environment variables override `settings.json` at launch:

| Variable | Description |
|---|---|
| `BROWSER_HARNESS_EXECUTOR_PROVIDER` | `azure` or `local` |
| `BROWSER_HARNESS_EXECUTOR_MODEL` | Executor deployment name or local model alias/ID |
| `BROWSER_HARNESS_EXECUTOR_ENDPOINT` | Executor endpoint URL |
| `BROWSER_HARNESS_PLANNER_PROVIDER` | `off`, `azure`, or `local` |
| `BROWSER_HARNESS_PLANNER_MODEL` | Planner deployment name or local model alias/ID |
| `BROWSER_HARNESS_PLANNER_ENDPOINT` | Planner endpoint URL |
| `BROWSER_HARNESS_PLANNER_REASONING` | Azure planner reasoning effort (`low`, `medium`, `high`) |
| `BROWSER_HARNESS_PLANNER_SCOPE` | `actions` (default) or `all` |
| `BROWSER_HARNESS_LOCAL_DOWNLOADS` | `1` permits downloading uncached local models |
| `BROWSER_HARNESS_LOCAL_MODEL` | Default local model alias for presets (default: `qwen2.5-1.5b`) |
| `BROWSER_HARNESS_AZURE_EXECUTOR` | Default Azure executor deployment for presets (default: `gpt-4.1-mini`) |
| `BROWSER_HARNESS_AZURE_PLANNER` | Default Azure planner deployment for presets (default: `gpt-4.1`) |
| `BROWSER_HARNESS_FOUNDRY_ENDPOINT` | Shared Azure AI Foundry HTTPS endpoint |
| `BROWSER_HARNESS_DEPLOYMENT` | Legacy alias for `BROWSER_HARNESS_EXECUTOR_MODEL` |
| `BROWSER_HARNESS_PLANNER` | Legacy alias for `BROWSER_HARNESS_PLANNER_MODEL` (`0` disables planner) |
| `BROWSER_HARNESS_SESSION` | `playwright-cli` session name (default: `browser-harness`) |
| `BROWSER_HARNESS_SKILLS` | `1` exposes the bundled `playwright-cli` skill via the `skill` tool |

---

## Terminal Interface & Controls

### Slash Commands & Buttons

| Command / Button | Action |
|---|---|
| `/help`, `F1` | Display usage examples, available commands, and active log/workspace paths |
| `/models`, **Models** | Open the centered model configuration and diagnostic modal |
| `/login <url>` | Open a URL in the persistent browser profile so you can sign in manually |
| `/teach <task>` | Start demonstration recording; any message typed while recording is saved as a step note |
| `/teach` *(while recording)* | Stop recording, synthesize the semantic guide, and save to `playbooks/` |
| `/teach cancel` | Discard the current recording without saving |
| `/playbooks` | List all taught playbooks and user rules |
| `/workspace` (`/files`) | List files saved in the current session workspace and recent sessions |
| `/usage` | Show per-model call counts, input/cached/output tokens, and reported cost |
| `/memory` | Display the current long-term navigation memory (`memory/long_term.md`) |
| `/clear` (`/new`, `/reset`) | Reset conversation context, plan, and usage counters, and start a fresh workspace folder |
| `/restart` | Reset session context and close the browser window |
| `/close`, **Close browser** | Close the active browser session |
| `Ctrl+X`, **Stop** | Interrupt after the current operation and cancel remaining planned tasks |
| `/quit` (`/exit`), `Ctrl+Q` | Exit the application (automatically closes the browser session) |

### Keyboard & Mouse Shortcuts

- **`Enter`**: Submit prompt or command.
- **`Ctrl+J`** / **`Shift+Enter`** / **`Alt+Enter`**: Insert a newline in the composer without submitting.
- **`Tab`**: Auto-complete slash commands when the composer starts with `/`.
- **Right-click**: If text is selected in the conversation or composer, copies it to the clipboard; otherwise pastes clipboard text (or selected file paths on Windows) into the composer.

---

## How the Harness Works

### 1. Typed Task Plans & Intent Preservation

Every plan item is a validated object with a `text` description and a `kind`:
- `browser` — general navigation, search, or page interaction
- `research` — reading sources and verifying publication metadata
- `write` — saving structured outputs or summaries to the session workspace
- `send` — composing and submitting a message to a specific `recipient` (`expected_messages`, default `1`)
- `verify` — confirming that a sent message or state change is visible on the page
- `playbook` — updating a taught guide with a user correction

**Critical invariant:** A request to update or correct a playbook is always classified as `kind="playbook"`, never `kind="send"`, even if the rule text mentions sending messages. Maintenance and stop requests never authorize browser actions.

### 2. Allowlisted Browser Tool Commands

`browser_tool.py` validates every model tool call and invokes `playwright-cli` as an argument array without a shell. Unsafe commands (`eval`, `run-code`, cookie export, arbitrary flags) are never exposed to the model.

| Command | Behavior & Token Optimization |
|---|---|
| `read [url]` | Opens `url` (if given) and extracts main-region text (`<article>`, `<main>`, `[role=main]`, up to 8,000 chars) plus structured publication date, canonical URL, and article type in **one tool call**. |
| `find <text>` | Searches visible elements and returns matching roles, refs (`e25`), and link URLs. Supports `find A \| B` to test up to 4 alternatives (e.g. bilingual labels) in a single call. |
| `links` | Lists main-region headlines and URLs (`>= 25` chars) so the agent can `read <url>` directly. |
| `compose` | Enters a complete draft (up to 40,000 chars) into a verified chat textbox in safe chunks without sending. |
| `verify-message` | Confirms that the submitted draft appears once as a new message under the same recipient header and that the composer is now empty. |
| `click`, `fill`, `type`, `press`, `select`, `hover`, `check`, `uncheck` | Interact with page elements by `ref` or visible label. Returns **only the DOM lines that changed** after the action. |
| `open`, `goto`, `reload`, `go-back`, `go-forward`, `wait` | Navigation and settling helpers. After navigation, the harness waits for SPA DOM text length to stabilize. `wait <text>` polls both the main document and same-origin iframes. |
| `screenshot`, `snapshot` | `screenshot` attaches a viewport JPEG (~1k tokens) for visual recovery; `snapshot` returns a compacted accessibility tree (filtered to interactive controls on large pages). |

### 3. Transactional One-Message Composition

Standard browser automation often mangles long messages: tool arguments hit length caps, models split one report into multiple partial messages, or newline characters trigger premature `Enter` submissions in chat apps.

The harness enforces a four-step transaction for `send` tasks:
1. **Header & Composer Binding:** `compose` inspects both `context_ref` (the visible conversation header) and `args=[textbox_ref]`. If the visible header does not match `recipient`, or if the target is not an editable field, nothing is typed.
2. **Chunked Draft Entry:** The full draft (up to 40,000 characters) is typed in 2,000-character chunks using `Shift+Enter` for line breaks. The harness then reads back the composer value and verifies that the canonicalized draft matches the requested text character-for-character.
3. **Pre-Submit Re-Verification:** Immediately before `press Enter` or clicking a send button, the harness re-verifies that the draft text, recipient header, and page URL have not drifted, and blocks duplicate submissions.
4. **Post-Submit Evidence (`verify-message`):** The harness polls the conversation container for up to 10 seconds to confirm that the exact full-text occurrence count increased by `1` relative to the pre-compose baseline and that the composer is now empty. It records a cryptographic SHA-256 digest (`text_sha256`) as completion proof. If verification is ambiguous, the harness **never resends automatically**.

### 4. Dated Research & Source Verification

When a prompt requests dated coverage (e.g. *"AI news from today and yesterday, two from each day"*):
- `runtime_context.py` computes the authoritative local civil dates (`Today` and `Yesterday` in the host's timezone) and injects them into both the planner and executor prompts.
- `read` extracts publication timestamps from `meta[property="article:published_time"]`, `meta[name="datePublished"]`, `application/ld+json` (`Article` / `BlogPosting`), or single-article `<time datetime="...">` tags, converting timezone-aware timestamps to the local civil date.
- Listing pages, unverified URLs, and duplicate canonical URLs cannot satisfy article quotas.
- If fewer articles exist for a requested date, the harness reports the exact counts found and requires explicit user permission via `tasks action=ask` before proceeding with partial coverage.

### 5. Token Efficiency & Prompt-Cache Preservation

- **Append-Only Steps Between Compactions:** Because LLM providers cache identical prompt prefixes, the harness keeps the message list strictly append-only for `6` steps (or until uncompacted outputs exceed `16,000` characters).
- **Batched Compaction:** During compaction, older tool outputs are summarized to their URL, title, and saved workspace file path, while the most recent page reads (`KEEP_READS = 2`) and all reads from the immediately preceding step remain in full so multi-article synthesis never triggers redundant re-reads.
- **Workspace Read-Through:** If the model requests `read` on a URL it already read earlier in the turn after its text was compacted, the harness serves the saved text directly from `workspaces/<session-id>/pages/` without touching the browser.
- **Ad & Tracker Blocking:** `.playwright/cli.config.json` blocks common ad and push-notification domains, shrinking page snapshots and preventing overlay click interception.

### 6. Resilience & The Stuck Escalation Ladder

When browser actions fail or produce no visible DOM change, the harness escalates deterministically instead of looping blindly:
1. **Attempt 1 — Targeted Error Diagnosis:** Injects a specific recovery hint based on the Playwright error (`intercepts pointer events` → close overlay/banner or press `Escape`; `is not an <input>` → pick the `textbox` ref; `not found` → run `find` for a fresh ref).
2. **Attempt 2 — Automatic Visual Inspection:** Automatically captures a `screenshot` (if the model supports vision) so the model can see modals, spinners, or layout shifts.
3. **Attempts 3–4 — Structured Reflection:** Requires the model to state what it sees, why previous attempts failed, and two distinct alternative approaches before acting.
4. **Attempt 5+ — Human-in-the-Loop Escalation:** Directs the model to pause and ask the user via `tasks action=ask` rather than repeating failed actions or giving up prematurely.

---

## Generative Learning: Memory & Playbooks

### Two-Level Memory with Automatic Self-Review

1. **Session Memory (`memory/sessions/YYYYMMDD-HHMMSS.md`):** Written deterministically by Python code at the end of every turn (zero token cost). Logs the prompt, status, actions that worked, failures, per-model token usage, and **measured wasted steps** (e.g. model calls spent only on `tasks`, re-reading pages already in context, or taking 8 actions for a task that later succeeded in 3).
2. **Long-Term Memory (`memory/long_term.md`):** Injected into the system prompt. When a turn encounters failures, wasted steps, a new site, or a playbook refinement, a single tool-free model call consolidates the lesson into **at most one concise line per site** describing the shortest working route.
3. **Self-Review Guard:** When `long_term.md` reaches 15 lines, the harness automatically compacts it to at most 12 lines—and **rejects the compaction in code** if any previously known domain or URL host would be lost.

### Playbooks: Learning by Demonstration (`/teach`)

Instead of brittle RPA scripts that replay fragile CSS selectors, `/teach` records a human demonstration once and converts it into a **generative landmark map**:

```text
/teach check trip details on my airline site
   ... navigate and click in the Chromium window ...
   the return flight is under the second destination tab   <-- typed in TUI while recording = step note
/teach                                                     <-- stops recording and writes the guide
```

- All typed form values are automatically redacted (`<value>`) and URL query strings/fragments are stripped so personal data (names, booking codes, emails) is never stored.
- A short model call distills the recorded actions, visited URLs, and your verbatim step notes into a 10-line guide (`playbooks/<site>/<task>.md`) referencing exact visible labels in quotes.
- **Updating Rules:** When you tell the agent *"update the playbook: never open the profile page; send the invite directly from search results"*, the `playbooks` tool updates the rule under its topic ID (`replace` / same-topic replacement) and records your verbatim correction as authoritative.

---

## Session Workspace, Logs & Usage Reports

Every session writes runtime artifacts into Git-ignored local directories:

```text
workspaces/<session-id>/
  index.md             chronological log of prompts and saved files
  pages/01-<slug>.md   full text and metadata of every page read (automatic)
  shots/01.jpg         viewport screenshots inspected by the agent (automatic)
  typed.md             audit trail of text typed into browser fields
  *.md / *.txt / ...   summaries, reports, and drafts created via the `files` tool
```

To inspect token and call usage across sessions from `logs/*.jsonl`:

```powershell
.\.venv\Scripts\python.exe report.py            # all sessions, per model + grand totals
.\.venv\Scripts\python.exe report.py --last 5   # latest 5 sessions
```

---

## Browser Session & Manual Sign-In

The harness launches Chromium in headed mode using a dedicated persistent profile in `.profile/`.
- Using an isolated profile keeps the agent separated from your daily personal browser while preserving logins across harness runs.
- To sign in to an account (including MFA/SSO), run `/login <site>` (e.g. `/login linkedin.com`), sign in manually in the Chromium window, and then issue tasks in the TUI. The agent never stores or enters passwords.

---

## Security & Privacy

- **Zero Stored Credentials:** Azure AI Foundry uses `DefaultAzureCredential` (`az login`). `settings.json` rejects unknown fields, embedded URL credentials, and API keys.
- **Endpoint Validation:** Azure bearer tokens are only sent to validated HTTPS Azure AI Foundry / Azure OpenAI hostnames (`*.services.ai.azure.com`, `*.openai.azure.com`, `*.cognitiveservices.azure.com`). Foundry Local traffic is restricted strictly to HTTP loopback (`127.0.0.1`, `localhost`, `::1`).
- **Workspace Containment:** The `files` tool resolves paths against the session root (`Path.is_relative_to`), blocking directory traversal (`../`) and non-text file execution.
- **Untrusted Page Content:** Web page text is treated strictly as untrusted data, never as system instructions. Consequential actions require deterministic harness checks and inline user approval.

---

## Project Structure

| File / Directory | Purpose |
|---|---|
| `browser_harness.py` | CLI entry point; launches `BrowserApp` and ensures clean browser shutdown on exit |
| `tui.py` | Textual full-screen UI: chat timeline, live plan bar, inline approval/question panels, `/models` modal |
| `agent.py` | Core agent loop, deterministic safety/completion gates, context compaction, stuck ladder, memory hooks |
| `browser_tool.py` | `playwright-cli` wrapper, command allowlist, `read`/`links`/`find`, DOM change diffs, `compose` & `verify-message` |
| `model_config.py` | Credential-free settings (`settings.json`), presets, environment overrides, endpoint validation |
| `foundry.py` | Streaming OpenAI-compatible Chat Completions client for Azure AI Foundry and Foundry Local |
| `local_foundry.py` | Foundry Local CLI integration (`server status/start`, `model info/download/load`, loopback verification) |
| `planner.py`, `plan.py` | Optional planner client and typed `tasks` state machine (`browser`, `research`, `write`, `send`, `verify`, `playbook`) |
| `runtime_context.py` | Authoritative local civil time, relative-date quota parser, and article timestamp normalization |
| `memory.py` | Two-level memory (`memory/sessions/*.md` and `memory/long_term.md`), waste tracking, and self-review |
| `playbooks.py` | `/teach` demonstration recorder, PII redaction, semantic guide synthesis, and topic-based rule updates |
| `workspace.py` | Per-session sandboxed file workspace (`files` tool) |
| `usage.py`, `report.py` | Real-time per-model token/call accounting and CLI log reporter |
| `session_log.py` | Thread-safe JSONL event logger (`logs/YYYYMMDD-HHMMSS.jsonl`) |
| `clipboard.py`, `terminal_keys.py` | Windows clipboard copy/paste helpers and Windows Terminal `Shift+Enter` binding support |
| `skills.py`, `.claude/skills/` | Optional progressive disclosure reader for the bundled `playwright-cli` reference skill |
| `.playwright/cli.config.json` | Chromium configuration, `.profile` path, and ad/tracker domain blocklist |
| `settings.example.json` | Public, credential-free model configuration template |
| `assets/` | Embedded demo video (`Browser-Tool-Use-Harness-Demo.mp4`) |
| `tests/` | Offline unit and regression suites (`test_harness.py`, `test_agent_features.py`, `test_models.py`, `test_regressions.py`, `test_tui.py`) plus opt-in loopback Chromium integration (`test_browser_integration.py`) |

---

## Testing & Verification

The default test suite (174 offline tests) runs **100% offline** using fake models, synthetic DOM states, and temporary directories—it never contacts Azure, Foundry Local, or external websites:

```powershell
.\.venv\Scripts\python.exe -m unittest discover -s tests -v
```

To run the opt-in **real-browser integration test** (which starts a local loopback HTTP chat server and a temporary headless Chromium profile to verify multi-chunk Unicode draft composition, recipient checking, single-send verification, and duplicate-submit blocking):

```powershell
$env:BROWSER_HARNESS_INTEGRATION = "1"
.\.venv\Scripts\python.exe -m unittest discover -s tests -p test_browser_integration.py -v
Remove-Item Env:\BROWSER_HARNESS_INTEGRATION
```

---

## Public Sharing Checklist

All runtime state—browser profiles (`.profile/`), Playwright scratch files (`.playwright-cli/`), active settings (`settings.json`), logs (`logs/`), learned memory (`memory/`), taught playbooks (`playbooks/`), and session workspaces (`workspaces/`)—is excluded via `.gitignore`.

Before pushing to a public repository, verify that no local state or credentials are staged:

```powershell
git status --short
git ls-files
```

