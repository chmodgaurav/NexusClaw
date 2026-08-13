# NexusClaw

An agentic AI CLI tool for automating codebase tasks — locally via terminal or remotely via Telegram. Built on the [Vercel AI SDK](https://sdk.vercel.ai/), [Bun](https://bun.sh/), and [OpenRouter](https://openrouter.ai/).

---

## Architecture

```
nexusclaw/
├── index.ts                  # CLI entrypoint (commander)
├── ai/
│   ├── ai.config.ts          # OpenRouter model factory
│   └── index.ts
├── tui/
│   ├── wakeup.ts             # Banner + mode selector
│   └── terminal-md.ts        # marked-terminal renderer
└── modes/
    ├── cli.ts                # CLI sub-mode router (agent/plan/ask)
    ├── agent/                # Core agent engine
    │   ├── type.ts           # ActionLog, AgentConfig, ActionType types
    │   ├── action-tracker.ts # Append-only mutation log with status tracking
    │   ├── tool-executor.ts  # FS operations + overlay staging layer
    │   ├── agent-tools.ts    # AI SDK tool definitions (wraps executor)
    │   ├── approval.ts       # CLI interactive approval flow
    │   ├── diff-view.ts      # Unified diff generation
    │   └── orchestrator.ts   # Agent mode entry point
    ├── ask/
    │   └── orchestrator.ts   # Read-only Q&A agent, optional .md save
    ├── plan/
    │   ├── types.ts          # Plan, PlanStep types
    │   ├── planner.ts        # JSON-schema-constrained plan generator
    │   ├── selection.ts      # Multiselect step picker (CLI)
    │   ├── web-tools.ts      # Firecrawl web_search / web_crawl / fetch_url
    │   └── orchestrator.ts   # Plan mode entry point
    └── telegram/
        ├── index.ts             # Telegraf bot launch
        ├── handlers.ts          # Command + callback action registration
        ├── agent-run.ts         # ask/agent/planSteps runners for Telegram
        ├── approval-session.ts  # Inline keyboard approval flow
        ├── plan-session.ts      # Interactive plan toggle UI
        ├── auth.ts              # Owner-only guard
        ├── constants.ts         # Welcome message
        └── text.ts              # Telegram text helpers (clip, replyMd)
```

---

## How It Works

### Staging layer

All file mutations are **staged, never written directly**. `ToolExecutor` maintains an in-memory overlay (`Map<path, content>`) and a deletion set. `ActionTracker` records every operation as an `ActionLog` with status `pending | approved | rejected | executed`. Nothing hits disk until the user explicitly approves.

```
Agent calls tool → ToolExecutor stages change → ActionTracker logs it
                                                        ↓
                                            User reviews via approval flow
                                                        ↓
                                    applyApprovedFromTracker() writes to disk
```

### Modes

| Mode | Description | Mutations |
|---|---|---|
| **Agent** | Free-form agentic task on your codebase | Yes — with approval |
| **Ask** | Read-only Q&A with optional `.md` save | Optional file create |
| **Plan** | LLM generates a step plan → you pick steps → agent executes each | Yes — with approval |
| **Telegram** | All three modes via bot commands with inline keyboard approval | Yes — with approval |

---

## Prerequisites

- [Bun](https://bun.sh/) `>= 1.x`
- Node.js is **not** required (Bun runtime only)

---

## Installation

```bash
git clone https://github.com/chmodgaurav/NexusClaw.git
cd NexusClaw
bun install
```

---

## Configuration

Create a `.env` file in the project root:

```env
# Required
OPENROUTER_API_KEY=sk-or-...
OPENROUTER_DEFAULT_MODEL=anthropic/claude-3.5-sonnet

# Optional: enables web_search / web_crawl / fetch_url tools in Plan and Ask modes
FIRECRAWL_API_KEY=fc-...

# Required only for Telegram mode
TELEGRAM_BOT_TOKEN=...
TELEGRAM_OWNER_ID=...            # Your Telegram user ID (integer)

# Optional: custom skill directories (semicolon-separated)
SKILLS_DIRS=/path/to/skills;/another/path
```

> `TELEGRAM_OWNER_ID` gates all bot commands — only that user ID can interact with the bot.

`ai/ai.config.ts` builds the model from `OPENROUTER_API_KEY` and `OPENROUTER_DEFAULT_MODEL` via `@openrouter/ai-sdk-provider`; both are read at call time, so `OPENROUTER_DEFAULT_MODEL` must be set to a valid OpenRouter model ID.

---

## Usage

### Start

```bash
bun run index.ts wakeup
# or
bunx nexusclaw wakeup
```

You'll see the nexusclaw ASCII banner and a mode prompt.

### CLI mode

```
? Choose CLI sub-mode
❯ Agent Mode      — give the agent a task, review & apply changes
  Plan Mode       — generate a plan, pick steps, execute
  Ask Mode        — ask a question, get a markdown answer
  Back
```

### Telegram mode

```
/start              — show help
/ask <question>     — read-only Q&A about your codebase
/agent <task>       — agentic task with inline approval
/plan <goal>        — generate plan, toggle steps, execute
```

After `/agent` or `/plan`, the bot sends an approval message with:
- **Show Diff** — view unified diff of staged changes
- **Accept All** — write all staged changes to disk
- **Reject All** — discard everything

---

## Project Structure Notes

The repository also contains `ask.md` and `todo-list-app/` — these are example outputs generated by running NexusClaw's Ask and Agent modes against this project, kept as a demonstration of the tool's own capabilities rather than part of the tool itself.

---

## Future Improvements

- Persist Telegram approval sessions across bot restarts
- Configurable staging backend (currently in-memory only, lost on process exit)
- Additional web tool providers beyond Firecrawl

## Screenshots

_Add a screenshot of the CLI banner/mode picker and a Telegram approval flow here._

## Contributing

Issues and pull requests are welcome.

## License

See repository for license details.
