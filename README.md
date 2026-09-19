# calc-session-tokens

A zero-dependency toolkit and agent skill harness to accurately calculate, break down, and report token consumption across AI coding agent sessions—including **Google Antigravity (AGY)**, **OpenAI Codex**, and **Claude Code**.

It tracks parent sessions along with any invoked subagents, appropriately accounting for prompt caching, uncached inputs, model outputs, and reasoning/thinking tokens.

---

[English](README.md) | [日本語](README.ja.md)

---

## Key Features

- **Multi-Agent Support**: Unified token accounting across **Antigravity (AGY)**, **OpenAI Codex**, and **Claude Code**.
- **Instant Chat Invocation**: Simply enter `/session-tokens` in the IDE / agent chat box to receive a formatted Markdown breakdown.
- **Recursive Subagent Aggregation**: Automatically discovers and aggregates token usage for hierarchically spawned subagents (via `invoke_subagent`, `SubAgentActivity`, or Agent tools).
- **Accurate Cache & Billed Accounting**:
  - Distinguishes cached input (prompt cache hits) from newly processed uncached tokens.
  - Correctly categorizes reasoning / thinking tokens as output tokens rather than double-counting them.
  - Evaluates **Billed Total** (`Uncached Input + Output`) for standardized cost estimation across providers.
- **Zero External Dependencies**: Implemented entirely in Python standard library (includes a custom protobuf wire decoder for AGY SQLite databases without requiring `protobuf`).

---

## Usage

### Primary Method: Chat Invocation (Slash Command)

The easiest and recommended way is typing the slash command directly into the IDE / agent chat interface. The assistant will invoke the script and return a formatted Markdown report:

| Agent / IDE | Chat Command | Behavior |
|---|---|---|
| **Google Antigravity (AGY IDE / CLI)** | `/session-tokens` | Reports token consumption for the active conversation and its subagents. |
| **Claude Code** | `/session-tokens` | Reports tokens for the active Claude Code session and spawned Agent tools. |
| **OpenAI Codex** | `/codex-session-tokens` | Reports tokens for the active Codex thread and subagent activities. |

#### Sample Report Output in Chat

```markdown
# AGY Session Token Report
- **Conversation ID**: `a320c2da-27fd-...`
- **Detected Model**: `gemini-3.8-flash`
- **Total LLM Steps**: `14` (Parent: 14, Subagents: 0)

## 📊 Token Usage Summary (Total)
| Metric | Tokens | Remarks |
|---|---|---|
| **Prompt Cached Tokens** | **145,210** | Read from prompt cache |
| **Prompt Uncached Tokens** | **12,480** | New input tokens |
| **Output Tokens** | **3,120** | Generated tokens (including Thinking) |
| **Total Billed Tokens** | **15,600** | Effective billable volume (Uncached + Output) |
```

---

### CLI / Manual Execution (Advanced Options)

You can also run the underlying scripts directly from your terminal to target historical sessions or generate machine-readable JSON output:

#### 1. Google Antigravity (AGY)
```pwsh
# Analyze the latest active session
python .agents/skills/session-tokens/scripts/calc_session_tokens.py

# Specify a past conversation ID
python .agents/skills/session-tokens/scripts/calc_session_tokens.py --conversation-id "<CONVERSATION_ID>"

# Output in JSON format
python .agents/skills/session-tokens/scripts/calc_session_tokens.py --json
```

#### 2. Claude Code
```bash
# Analyze active session (auto-resolved from $CLAUDE_CODE_SESSION_ID or latest)
python .claude/skills/session-tokens/scripts/calc_session_tokens.py

# Specify a session ID
python .claude/skills/session-tokens/scripts/calc_session_tokens.py --session-id "<SESSION_ID>"

# Output in JSON format
python .claude/skills/session-tokens/scripts/calc_session_tokens.py --json
```

#### 3. OpenAI Codex
```pwsh
# Analyze current/latest session
python -X utf8 .agents/skills/codex-session-tokens/scripts/calc_codex_session_tokens.py

# Specify a thread ID
python -X utf8 .agents/skills/codex-session-tokens/scripts/calc_codex_session_tokens.py --session-id "<THREAD_ID>"

# Exclude spawned subagents (parent session only)
python -X utf8 .agents/skills/codex-session-tokens/scripts/calc_codex_session_tokens.py --no-subagents

# Output in JSON format
python -X utf8 .agents/skills/codex-session-tokens/scripts/calc_codex_session_tokens.py --json
```

---

## Token Metric Definitions

| Metric | Description |
|---|---|
| **Cached Input Tokens** | Tokens read from the prompt cache (often free or heavily discounted). |
| **Uncached Input Tokens** | Fresh input tokens processed by the model (`Input - Cached`). |
| **Output Tokens** | Generated response tokens, including reasoning / thinking tokens. |
| **Total Billed Tokens** | Effective billable token volume (`Uncached Input + Output`). |
| **All Processed Tokens** | Total tokens handled by the context window (`Input + Output`). |

---

## Supported Agents & Implementation Details

| Agent / Harness | Skill Directory | Target Log / DB Format |
|---|---|---|
| **Google Antigravity (AGY)** | `.agents/skills/session-tokens` | SQLite databases (`conversations/*.db`) + `transcript.jsonl` |
| **Claude Code** | `.claude/skills/session-tokens` | Project JSONL logs (`~/.claude/projects/*/*.jsonl`) |
| **OpenAI Codex** | `.agents/skills/codex-session-tokens` | Rollout logs (`~/.codex/sessions/**/*.jsonl`) |

---

## Project Structure

```
calc-session-tokens/
├── .agents/
│   └── skills/
│       ├── session-tokens/              # Antigravity (AGY) skill definition
│       │   ├── SKILL.md
│       │   └── scripts/
│       │       └── calc_session_tokens.py
│       └── codex-session-tokens/        # OpenAI Codex skill definition
│           ├── SKILL.md
│           └── scripts/
│               └── calc_codex_session_tokens.py
├── .claude/
│   └── skills/
│       └── session-tokens/              # Claude Code skill definition
│           ├── SKILL.md
│           └── scripts/
│               └── calc_session_tokens.py
├── LICENSE                              # MIT License
├── README.md                            # English Documentation
└── README.ja.md                         # Japanese Documentation
```

---

## Requirements

- **Python 3.8+**
- No third-party packages required (standard library only).

---

## License

This project is licensed under the [MIT License](LICENSE).
