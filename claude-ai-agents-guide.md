# The 4 Types of Claude AI Agents
### Complete Guide + Architecture (Part 1)

Everyone says "build AI agents in Claude" — but nobody explains what an agent actually *is* or how it works under the hood. This guide breaks down the 4 foundational agent architectures you can set up in Claude Code today, from basic to advanced.

**What is an AI Agent?**
An agent = an AI model (Claude) + tools + a loop. Instead of just answering your question, an agent can *take actions* — read your calendar, update a database, send an email — and keep working autonomously until the task is done.

```
┌─────────────────────────────────────────────┐
│                THE AGENT LOOP               │
│                                             │
│   Your Goal ──► Claude thinks ──► Uses a    │
│                    ▲               tool     │
│                    │                │       │
│                    └── observes ◄───┘       │
│                        result               │
│                                             │
│   Loop repeats until the task is complete   │
└─────────────────────────────────────────────┘
```

---

## Type 1: Basic Agent with Tools

**What it is:** The simplest agent. You give Claude access to specific tools/APIs — like Gmail or Google Calendar — and it autonomously plans and executes your tasks using them.

**Architecture:**

```
┌──────────┐      ┌─────────────────┐      ┌──────────────────────┐
│   USER   │─────►│  CLAUDE (Brain)  │─────►│  TOOLS               │
│ "Plan my │      │  • Understands   │      │  ├─ Google Calendar  │
│  week"   │      │  • Decides which │      │  ├─ Gmail API        │
│          │◄─────│    tool to call  │◄─────│  └─ Weather API      │
└──────────┘      └─────────────────┘      └──────────────────────┘
     Result           Reasoning loop            Real-world actions
```

**How it works:**
1. You define tools (each tool = a name + description + input schema)
2. Claude reads your request and decides which tool to call and with what inputs
3. The tool executes (e.g., fetches calendar events)
4. Claude reads the result and either calls another tool or gives you the final answer

**Setup (Anthropic API — tool definition):**

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_calendar_events",
        "description": "Fetch events from the user's Google Calendar for a date range",
        "input_schema": {
            "type": "object",
            "properties": {
                "start_date": {"type": "string", "description": "YYYY-MM-DD"},
                "end_date": {"type": "string", "description": "YYYY-MM-DD"}
            },
            "required": ["start_date", "end_date"]
        }
    }
]

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "What does my week look like? Find me 2 free hours for deep work."}]
)
# Claude responds with a tool_use block → your code calls the real Calendar API
# → you return the result → Claude plans your week autonomously
```

**Best for:** Personal assistants, schedulers, single-purpose automations
**Difficulty:** ⭐ Beginner

---

## Type 2: Agent with MCP Servers

**What it is:** The most powerful connection method. MCP (Model Context Protocol) is an open standard that lets your agent talk *directly* to external systems — Notion, GitHub, Slack, Postgres, or any database — through standardized "servers." Think of MCP as a USB-C port for AI: one protocol, unlimited integrations.

**Architecture:**

```
                        ┌──────────────────────┐
                        │   CLAUDE (MCP Client) │
                        └──────────┬───────────┘
                                   │  Model Context Protocol
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
     ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
     │  NOTION MCP    │   │  GITHUB MCP    │   │  DATABASE MCP  │
     │  SERVER        │   │  SERVER        │   │  SERVER        │
     │  • read pages  │   │  • read repos  │   │  • run queries │
     │  • create docs │   │  • create PRs  │   │  • insert rows │
     └───────┬────────┘   └───────┬────────┘   └───────┬────────┘
             ▼                    ▼                    ▼
        Notion API           GitHub API           PostgreSQL
```

**How it works:**
1. Each MCP server exposes tools + resources for one system (Notion, GitHub, etc.)
2. Claude discovers the available tools automatically — no custom glue code
3. Your agent can chain actions ACROSS systems: read a GitHub issue → summarize it → log it in Notion

**Setup (Claude Code — add an MCP server in one command):**

```bash
# Add the Notion MCP server
claude mcp add notion -- npx -y @notionhq/notion-mcp-server

# Add the GitHub MCP server
claude mcp add github -- npx -y @modelcontextprotocol/server-github

# List connected servers
claude mcp list
```

Or configure via `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": { "NOTION_TOKEN": "your_token_here" }
    }
  }
}
```

Now just prompt: *"Read my 'Content Ideas' Notion database and create a GitHub issue for each video idea marked 'Approved'."* — Claude handles the rest.

**Best for:** Connecting to SaaS tools, databases, company knowledge bases
**Difficulty:** ⭐⭐ Beginner–Intermediate

---
## Type 3: Sequential Agent (Pipeline)

**What it is:** An assembly line of agents. Each agent does ONE job well, then passes its output to the next agent. Example from the video: Agent A reads your contacts → Agent B formats the info → Agent C sends the emails.

**Architecture:**

```
 INPUT                 THE ASSEMBLY LINE                      OUTPUT
┌───────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│Contact│──►│  AGENT A    │──►│  AGENT B    │──►│  AGENT C    │──► Emails
│ List  │   │  READER     │   │  FORMATTER  │   │  SENDER     │    Sent ✅
└───────┘   │ Extracts    │   │ Writes      │   │ Sends via   │
            │ names/emails│   │ personalized│   │ Gmail API + │
            │ from CSV/CRM│   │ email drafts│   │ logs status │
            └─────────────┘   └─────────────┘   └─────────────┘
                 Output of each step = Input of the next step
```

**How it works:**
1. Break a complex task into clear stages
2. Each stage gets its own focused prompt (and only the tools it needs)
3. Output of Step 1 becomes input of Step 2 — a deterministic, debuggable chain
4. If one stage fails, you know exactly where — much easier than one giant prompt

**Setup (Python — a simple 3-stage pipeline):**

```python
import anthropic
client = anthropic.Anthropic()

def run_agent(system_prompt, user_input):
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=2048,
        system=system_prompt,
        messages=[{"role": "user", "content": user_input}]
    )
    return response.content[0].text

# STAGE 1: Reader Agent
contacts = run_agent(
    "You are a data extraction agent. Extract name, email, company from raw text. Output clean JSON only.",
    raw_contact_data
)

# STAGE 2: Formatter Agent
drafts = run_agent(
    "You are an email copywriter agent. For each contact in this JSON, write a personalized outreach email. Output JSON: [{email, subject, body}].",
    contacts
)

# STAGE 3: Sender Agent (uses a send_email tool)
result = run_agent(
    "You are a dispatch agent. Validate each draft and call the send_email tool for each one. Report a summary.",
    drafts
)
```

**In Claude Code**, you can achieve the same with **subagents** — create them in `.claude/agents/` and Claude delegates each stage:

```markdown
# .claude/agents/formatter.md
---
name: formatter
description: Formats extracted contact data into personalized email drafts
tools: Read, Write
---
You take raw contact JSON and produce polished, personalized email drafts...
```

**Best for:** Multi-step workflows — outreach pipelines, content production, data ETL
**Difficulty:** ⭐⭐⭐ Intermediate

---

## Type 4: Parallel Execution Agent

**What it is:** Multiple agents working *simultaneously* on different parts of a task. At the end, an orchestrator merges all their outputs into one final result. This is how you make agents FAST.

**Architecture:**

```
                      ┌─────────────────────┐
                      │  ORCHESTRATOR AGENT │
                      │  Splits the task    │
                      └─────────┬───────────┘
             ┌─────────────────┼─────────────────┐
             ▼ (same time)     ▼ (same time)     ▼ (same time)
      ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
      │  AGENT 1    │   │  AGENT 2    │   │  AGENT 3    │
      │  Researches │   │  Analyzes   │   │  Scrapes    │
      │  competitors│   │  your data  │   │  reviews    │
      └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
             └─────────────────┼─────────────────┘
                               ▼
                      ┌─────────────────────┐
                      │    MERGE AGENT      │
                      │  Combines all data  │
                      │  → Final Report ✅  │
                      └─────────────────────┘
```

**How it works:**
1. An orchestrator splits one big task into independent subtasks
2. Each subtask runs on its own agent, in parallel (not one-by-one)
3. A final merge step synthesizes everything into one output
4. 3 agents × 1 minute each = ~1 minute total (instead of 3 minutes sequentially)

**Setup (Python — asyncio parallel agents):**

```python
import asyncio
import anthropic

client = anthropic.AsyncAnthropic()

async def run_agent(system_prompt, task):
    response = await client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=2048,
        system=system_prompt,
        messages=[{"role": "user", "content": task}]
    )
    return response.content[0].text

async def main():
    # All 3 agents fire AT THE SAME TIME
    results = await asyncio.gather(
        run_agent("You are a market research agent.", "Research top 5 competitors of X"),
        run_agent("You are a data analyst agent.", "Analyze this sales CSV: ..."),
        run_agent("You are a review analysis agent.", "Summarize these customer reviews: ..."),
    )

    # MERGE step: one agent combines everything
    final_report = await run_agent(
        "You are a synthesis agent. Merge these 3 reports into one executive summary.",
        "\n\n---\n\n".join(results)
    )
    print(final_report)

asyncio.run(main())
```

**In Claude Code**, just ask: *"Use 3 parallel subagents — one to audit the frontend, one the backend, one the tests — then merge findings into REPORT.md."* Claude Code spawns parallel subagents, each with its own context window.

**Best for:** Research at scale, codebase audits, anything with independent subtasks
**Difficulty:** ⭐⭐⭐⭐ Advanced

---

## Quick Comparison

| Type | Pattern | Speed | Complexity | Best Use Case |
|------|---------|-------|------------|---------------|
| 1. Tools Agent | Claude + APIs | Fast | ⭐ | Calendar/email assistant |
| 2. MCP Agent | Claude + MCP servers | Fast | ⭐⭐ | Notion, GitHub, databases |
| 3. Sequential | A → B → C pipeline | Slower, reliable | ⭐⭐⭐ | Multi-step workflows |
| 4. Parallel | Split → run together → merge | Fastest for big tasks | ⭐⭐⭐⭐ | Research & audits at scale |

**Pro tip:** Real production systems COMBINE these — e.g., parallel agents (Type 4) where each worker uses MCP servers (Type 2) inside a sequential pipeline (Type 3).

---

### 🔥 Part 2 Coming Soon
The 3 most dangerous, high-level agent architectures — orchestrator swarms, self-correcting evaluator loops, and long-running autonomous agents — are coming in Part 2. Follow so you don't miss it.
