# Thread skills

Open-source [agent skills](https://agentskills.io) for [Thread](https://jointhread.com) —
the scoped, read-only mail feed for AI agents. Install a skill and your agent knows
how to read a user's email through Thread: one API key, one endpoint to loop on, and
it only ever sees the mail the user granted.

Built for agents like **[Hermes](https://hermes-agent.nousresearch.com)** and
**OpenClaw**, and works with any agent that reads the [agentskills.io](https://agentskills.io)
`SKILL.md` standard (Claude Code, Cursor, and others).

## Skills

| Skill | What it does |
|-------|--------------|
| [`thread-mail`](skills/thread-mail/) | Read a user's mail: triage an inbox, watch for new messages, pull a receipt, download an attachment. Read-only, pre-scoped, privacy-preserving. |

## Install

Each skill is a self-contained folder under `skills/`. A `SKILL.md` (with YAML
frontmatter) is the entry point; a `references/` folder holds the full API contract
the skill points to.

**Hermes** — publish/import via the Skills Hub, or drop the skill folder into your
agent's skills directory.

**OpenClaw / Claude Code / Cursor** — copy the skill folder into your skills
directory (e.g. `.claude/skills/thread-mail/`), or clone this repo and point your
agent's skills path at `skills/`.

```bash
git clone https://github.com/jointhread/thread-skills
cp -r thread-skills/skills/thread-mail ~/.claude/skills/
```

Then give the agent two values:

- `THREAD_API_KEY` — mint one at your Thread dashboard → **Agent keys**. Looks like `thread_sk_…`.
- `THREAD_BASE_URL` — your Thread gateway host (`http://localhost:3002` in dev).

## How Thread works, in one paragraph

A user connects their mailbox to Thread once and mints a **scoped agent key** —
choosing exactly which mail an agent may read (by category, sender, subject; with or
without message bodies and attachments). The agent calls one endpoint, `GET /emails`,
and gets back only the mail inside that grant, already filtered. Out-of-scope mail is
simply absent — the agent never sees the rest of the inbox and never handles a
permission error. Raw message bodies never leave Thread; bodies are masked or absent.

## Contributing

Skills follow the [agentskills.io](https://agentskills.io) standard: a `SKILL.md`
with `name` + `description` frontmatter, and any supporting files it references. Keep
claims about the API in sync with the live gateway spec (`GET {BASE}/openapi.json`) —
the reference docs here are rendered copies of it.

## License

MIT — see [LICENSE](LICENSE).
