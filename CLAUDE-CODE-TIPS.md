## Token Management Tips for Claude Code

### How Tokens Actually Work (The Foundation)
Claude rereads the **entire conversation history** on every message — so costs compound exponentially, not linearly. A message 30 turns in could cost 31x more than message 1. On top of that, your CLAUDE.md, MCP servers, system prompts, and loaded files reload on every single turn as invisible overhead.

---

### Tier 1 — Easy Wins

- **Start fresh conversations** — use `/clear` between unrelated tasks. This is the single biggest lever since every message in a long chat is exponentially more expensive.
- **Disconnect unused MCP servers** — one server alone can add ~18,000 tokens per message. Prefer CLI tools over MCPs where possible.
- **Batch your prompts** — three separate messages cost roughly 3x a single combined message. Also, edit your original message to correct mistakes rather than sending follow-ups, since follow-ups stack permanently onto history.
- **Use plan mode before coding** — prevents Claude from going down the wrong path and burning tokens on work you'll scrap. Add a rule like *"don't make changes until 95% confident"* to your CLAUDE.md.
- **Run `/context` and `/cost`** — makes the invisible visible. You'll see exactly what's eating your tokens (MCP overhead, loaded files, history).
- **Set up a status line** in your terminal to monitor your context percentage in real time.
- **Keep your usage dashboard open** and pace yourself — or automate an alert when you're nearing your limit.
- **Be surgical with what you paste** — if the bug is in one function, paste only that function, not the whole file.
- **Watch Claude work** — don't walk away on long tasks. Catch it going down a wrong path early before it wastes thousands of tokens in a loop.

---

### Tier 2 — Intermediate

- **Keep CLAUDE.md lean (under 200 lines)** — it's reread on every single message. Treat it as an index that points to where data lives, not a place that stores the data itself.
- **Be surgical with file references** — instead of "here's the whole repo, find the bug," say "check the `verifyUser` function in `auth.js`." Use `@filename` to point at specific files.
- **Compact manually at ~60% context** — don't wait for the 95% auto-compact trigger; by then context quality has already degraded. After 3–4 compacts, do a session summary + `/clear` and restart.
- **Beware the 5-minute cache timeout** — Claude uses prompt caching to avoid reprocessing unchanged context, but it expires after 5 minutes. If you step away and come back, your next message reprocesses everything from scratch at full cost. Run `/compact` or `/clear` before stepping away.
- **Control command output bloat** — when Claude runs shell commands, the full output enters your context window. Deny permissions for commands a project doesn't need.

---

### Tier 3 — Advanced

- **Pick the right model** — Sonnet for most coding, Haiku for sub-agents and simple tasks, Opus sparingly (target under 20% of usage) for deep architectural planning only.
- **Understand sub-agent costs** — agent workflows use roughly 7–10x more tokens than a single-agent session because each sub-agent wakes up with its own full context reload. Use them for isolated one-off tasks, and spawn sub-agents in Haiku to keep those costs lower.
- **Schedule around peak hours** — peak is 8am–2pm ET on weekdays, when your session drains faster. Save big refactors and multi-agent sessions for afternoons, evenings, or weekends.
- **Time your sessions against your reset** — if you're near a reset with budget remaining, go heavy and get your money's worth. If you're near your limit with lots of time left, step away and come back with a full budget.
- **Make CLAUDE.md a living system constitution** — store stable architectural decisions and rules (like "spawn sub-agents for any task needing 3+ files") so every future prompt gets shorter. The idea: save decisions, not conversations.

---

### The Core Mindset Shift
Most token problems aren't a plan size problem — they're a **context hygiene problem**. Stop resending your entire conversation history dozens of times when you could manage it down to a fraction with these habits.
