# Agent Loop — build, play, fix

A Google ADK **orchestrator** hands five framework workers a shared goal: build a small language-learning arcade. They write games in parallel onto one site. Then a **QA agent** opens each game in a real Chrome window (Playwright MCP), clicks through it, and reports WORKS or BROKEN. Broken games get **one** bounded fix round. Open `site/index.html` to play.

This is the Week 5 capstone from [Ed Donner’s Agents course](https://github.com/ed-donner/agents), packaged as a standalone repo. The loop that matters for product/QA work: **requirement → build → E2E check in a browser → fail with a symptom → one repair**.

**Not a language-learning product.** The arcade is the sandbox. The skill is judging agent-built work the way you would judge an “AI Employee” that just finished a task.

## What I tried to learn

| Idea | How it shows up here |
|---|---|
| **Outer agent loop** | The orchestrator is an agent, not a hardcoded `for` loop. It decides who builds, what to test, and who to send back. |
| **E2E QA on agent output** | `test_game` plays `game.html` in Chrome and writes a verdict (`works` + one-sentence note). |
| **Edge cases** | Missing worker files, hung builders (timeout + kill), rate limits, no Chrome → file-existence fallback. |
| **Translate a goal into checks** | Each worker gets a learning objective; QA judges that objective in the browser, not “did the LLM reply.” |
| **MCP** | Filesystem MCP for builders; Playwright MCP for QA. |
| **Observability** | Live SQLite board (plan / in progress / done) painted in the terminal while workers run. |

## The architecture

```mermaid
flowchart TB
  you[You: language + team]
  orch[ADK orchestrator]
  style[author_style]
  build[launch_worker x N]
  wait[wait_for_team]
  qa[QA agent + Playwright MCP]
  fix[relaunch_worker once]
  hub[build_hub]
  site[site/index.html]

  you --> orch
  orch --> style
  orch --> build
  build --> wait
  wait --> qa
  qa -->|BROKEN| fix
  fix --> wait
  qa -->|WORKS| hub
  hub --> site
```

**Layers**

1. **Orchestrator** (`orchestrator.py`) — ADK tools: style, launch, wait, test, one fix, hub.
2. **Builders** — same worker shape in five frameworks (Strands, Pydantic AI, Microsoft Agent Framework, Agno, Mastra). Each claims one task on a shared SQLite board and writes `site/<slug>/`.
3. **QA** (`qa_agent.py`) — fresh, short-lived ADK agent per game, Playwright MCP, call budget, timeout.
4. **Output** — static `site/` (gitignored).

Skip a framework with `--skip mastra` if you do not want Node.

## MCP

| Who | Server | Why |
|---|---|---|
| Each builder | filesystem MCP (`npx` / stdio) | Read/write only inside the shared `site/` (or its own `workspace/` when run solo) |
| QA agent | `@playwright/mcp` | Open `game.html`, click, watch console |

If Chrome or the Playwright server is missing, QA falls back to “are the files there?” and still prints the site URL.

## AI

- **Orchestrator + QA:** Google ADK / Gemini (`ORCHESTRATOR_MODEL`, default `gemini-3.1-flash-lite`)
- **Builders:** OpenAI-compatible (`WORKER_MODEL`, default `gpt-5.4-mini`)
- **Pattern:** orchestrator owns decisions; tools own mechanics (subprocess launch, browser teardown)

## Observability

| What | Where |
|---|---|
| Plan / steps / done | Shared `board.sqlite`, live terminal board (`live_board.py`) |
| QA verdict | stdout: `WORKS` / `BROKEN` + note |
| Incomplete game | Final check lists missing `game.html` / assets |
| Hung worker | Wait tool kills the process after a time limit |

There is no APM. The board + verdicts are the audit trail of the run.

## Setup

Python 3.12+, [uv](https://docs.astral.sh/uv/), Node (for Mastra + Playwright MCP), Google Chrome.

```bash
git clone https://github.com/saaragmon/agent-loop.git
cd agent-loop
cp .env.example .env
# GOOGLE_API_KEY and OPENAI_API_KEY
uv sync
cd workers/mastra && npm install && cd ../..
```

## Run

```bash
uv run agent_loop.py
```

Watch the board fill in. When it finishes, open the printed `site/index.html`.

```bash
uv run agent_loop.py --language French
uv run agent_loop.py --skip mastra agno
uv run agent_loop.py --dry-run
uv run agent_loop.py --no-open
```

Stop with Ctrl+C if a run is stuck spending API credits.

## Layout

```
agent_loop.py      CLI
orchestrator.py    ADK agent + tools
qa_agent.py        Playwright QA sub-agent
catalog.py         Which workers exist and how to launch them
workers/           One folder per framework
site/              Generated arcade (gitignored)
```

## License

Personal coursework / portfolio. Not affiliated with the course publisher.
