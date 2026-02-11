# Cloudflare Starter Skill

**Deploy fullstack web apps on Cloudflare — guided by your AI coding agent.**

An open skill that teaches any AI coding agent how to build and deploy fullstack applications on Cloudflare's edge network. Free tier, no credit card, scales to zero.

Built for students and beginners. You make the decisions — the AI executes.

Works with: Claude Code, Cursor, Windsurf, Codex, GitHub Copilot, Gemini CLI, Roo Code, and [25+ other tools](https://agentskills.io).

---

## What Is a "Skill"?

AI coding agents (Claude Code, Cursor, Copilot, etc.) are powerful, but they have a fundamental limitation: **they can only work with what fits in their context window**. That's the working memory the AI uses during a conversation — typically between 100K and 1M tokens depending on the model.

A real-world codebase, its documentation, and all the knowledge needed to deploy it correctly can easily exceed that limit. And even when things do fit, stuffing everything in at once makes the AI *worse* — irrelevant information competes for attention and degrades output quality.

### The problem

Imagine asking your AI agent to "deploy this app to Cloudflare." Without a skill, the agent has to:

1. Rely on training data that may be months or years out of date
2. Guess at current API patterns, CLI flags, and best practices
3. Search the web and hope it finds the right docs
4. Hold all of that in memory alongside your actual codebase

The result: hallucinated commands, outdated patterns, and a lot of back-and-forth.

### How skills solve it

A **skill** is a folder of markdown files that gives an AI agent specialized, up-to-date knowledge — loaded on demand, not all at once.

```
cloudflare-starter-skill/
├── SKILL.md              ← Agent reads this first (overview, decision matrix, rules)
└── references/
    ├── getting-started.md    ← Loaded only when setting up Cloudflare
    ├── database-storage.md   ← Loaded only when adding a database
    ├── auth-security.md      ← Loaded only when adding login
    └── ...8 more files       ← Each loaded only when relevant
```

This pattern is called **progressive disclosure**:

1. **Startup**: The agent sees only the skill name and description (a few tokens)
2. **Task match**: When the user asks about Cloudflare, the agent loads `SKILL.md` (~170 lines)
3. **Deep dive**: When the agent needs database setup, it loads *just* `database-storage.md`

The agent never holds all 10 reference files in memory at once. It pulls in exactly what it needs, when it needs it — keeping context focused and output quality high.

### Cross-tool compatibility

This skill follows the [**Agent Skills** open standard](https://agentskills.io) — an open format originally developed by Anthropic and now adopted by 25+ tools across the industry.

Every major AI coding tool has its own instruction file convention:

| Tool | Native format | Reads skills? |
|------|--------------|---------------|
| **Claude Code** | `CLAUDE.md` + `SKILL.md` | Native support |
| **GitHub Copilot** | `.github/copilot-instructions.md` | [Via Agent Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills) |
| **Cursor** | `.cursor/rules/*.mdc` | Via `SKILL.md` |
| **Windsurf** | `.windsurfrules` | Via `SKILL.md` |
| **Codex CLI** | `AGENTS.md` | Via `AGENTS.md` (included) |
| **Gemini CLI** | `GEMINI.md` / `AGENTS.md` | Via `AGENTS.md` (included) |
| **Roo Code / Cline** | `.clinerules` / `AGENTS.md` | Via `AGENTS.md` (included) |

This repo includes both `SKILL.md` (the standard) and `AGENTS.md` (a copy, for tools that look for that filename). No tool-specific config dirs needed.

---

## Why Cloudflare?

There are many ways to deploy a web app. Vercel, Netlify, AWS, GCP, Fly.io, Railway — all solid options. We chose Cloudflare for this skill because it hits a unique sweet spot for students and beginners:

**It's free.** Not "free trial" or "free for 30 days." The free tier includes compute, database, file storage, caching, queues, AI inference, and bot protection — with no credit card required. You can build and deploy a complete fullstack app without spending anything.

**Everything is in one place.** Most providers make you stitch together services from different platforms (e.g., Vercel for frontend + Supabase for database + S3 for files). Cloudflare provides all of it under one account, one CLI, one billing page.

**Everything is CLI-controlled.** A single tool (`wrangler`) handles creating databases, deploying code, managing secrets, running migrations, and tailing logs. No clicking through dashboards. This matters because AI coding agents work through the terminal — a CLI-first platform means the AI can do everything for you.

**It scales to zero.** You pay nothing when nobody's using your app. No idle servers, no minimum charges.

| Need | Cloudflare Service | Free Tier |
|------|-------------------|-----------|
| API / serverless functions | **Workers** | 100K req/day |
| Frontend SPA or SSR | **Workers + Static Assets** | Unlimited bandwidth |
| Relational database (SQL) | **D1** (SQLite) | 5M reads/day, 5GB |
| File/image/video storage | **R2** (S3-compatible) | 10GB, zero egress |
| Sessions, config, cache | **KV** (key-value) | 100K reads/day |
| User authentication | **Better Auth** (OSS library) | Free |
| Background jobs | **Queues** | 10K ops/day |
| AI inference | **Workers AI** | 10K neurons/day |
| Bot/spam protection | **Turnstile** | 1M req/month |

> **Note:** The patterns in this skill (domain-driven structure, progressive disclosure, ARCHITECTURE.md) work with *any* provider. If you later switch to Vercel or AWS, the architectural knowledge transfers.

---

## Architecture at a Glance

```
┌──────────────────────────────────────────────────┐
│              Cloudflare Edge (300+ DCs)           │
│                                                    │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │  Workers    │  │  Static    │  │  Workers AI  │  │
│  │  (API/SSR)  │  │  Assets    │  │  (inference) │  │
│  └─────┬──────┘  └────────────┘  └─────────────┘  │
│        │                                           │
│  ┌─────┴──────┐  ┌────────────┐  ┌─────────────┐  │
│  │  D1 (SQL)  │  │  R2 (S3)   │  │  KV (cache)  │  │
│  └────────────┘  └────────────┘  └─────────────┘  │
└──────────────────────────────────────────────────┘
```

Everything runs at the edge (300+ data centers). No servers to manage.

---

## What's Inside

| File | What it covers |
|------|----------------|
| [`SKILL.md`](SKILL.md) | Main skill file — quick start, service matrix, architecture rules, agent execution flow |
| [`references/beginner-onboarding.md`](references/beginner-onboarding.md) | Brand new to web dev? Start here. Explains everything from scratch + project ideas |
| [`references/getting-started.md`](references/getting-started.md) | Cloudflare account setup, wrangler install, first deploy |
| [`references/tutorial-todo-app.md`](references/tutorial-todo-app.md) | Build a complete fullstack todo app step-by-step (the best starting point) |
| [`references/compute.md`](references/compute.md) | Workers, Python Workers, containers, runtime limits |
| [`references/database-storage.md`](references/database-storage.md) | D1 (SQL), R2 (files), KV (cache), Queues (jobs) |
| [`references/auth-security.md`](references/auth-security.md) | Better Auth, JWT, Turnstile, access control |
| [`references/deployment-cicd.md`](references/deployment-cicd.md) | CLI deploy, GitHub Actions, secrets management |
| [`references/js-patterns.md`](references/js-patterns.md) | Hono API, Drizzle ORM, React + Vite frontend patterns |
| [`references/python-patterns.md`](references/python-patterns.md) | FastAPI on Workers via Pyodide WASM |
| [`references/architecture-bestpractices.md`](references/architecture-bestpractices.md) | Project structure, state management, security, ARCHITECTURE.md template |
| [`references/debugging-logging.md`](references/debugging-logging.md) | Structured logging, error tracking, wrangler tail |

---

## Setup by Tool

### Claude Code

```bash
# Install the skill globally
claude skill add --global SerjoschDuering/cloudflare-starter-skill
```

Or clone locally:

```bash
git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .claude/skills/cloudflare
```

### Cursor / Windsurf

Clone into your project — the tool picks up `SKILL.md` automatically:

```bash
git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .skills/cloudflare
```

### Codex / Gemini CLI / Cline / Roo Code

These tools read `AGENTS.md` (included as a mirror of `SKILL.md`):

```bash
git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .skills/cloudflare
```

### Any Other Tool

It's just markdown. Clone and point your tool at the files:

```bash
git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git
```

---

## Quick Start

Once the skill is loaded, just tell your AI agent what you want to build:

> "I want to build a todo app on Cloudflare"

> "Help me deploy a fullstack React app with a database"

> "I have no idea what to build — give me project ideas"

The agent reads the right reference files and guides you through every step — from account setup to production deployment.

---

## Recommended Stacks

### JavaScript/TypeScript

| Layer | Tool |
|-------|------|
| API | Hono (Workers-native, fully typed) |
| ORM | Drizzle (D1 support, generates Zod validators) |
| Frontend | React + Vite |
| Routing | TanStack Router (file-based) |
| Server state | TanStack Query |
| Client state | Zustand |
| Auth | Better Auth |

### Python

| Layer | Tool |
|-------|------|
| API | FastAPI (runs on Workers via Pyodide WASM) |
| CLI | `pywrangler` (wraps wrangler, uses `uv`) |

---

## Further Reading

### Agent Skills Standard
- [Agent Skills specification](https://agentskills.io/specification) — the open standard this skill follows
- [Agent Skills GitHub repo](https://github.com/agentskills/agentskills) — spec source code
- [Anthropic: Equipping agents with Agent Skills](https://www.anthropic.com/news/agent-skills) — original announcement
- [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — why context management matters

### Tool Documentation
- [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- [VS Code Agent Skills support](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
- [GitHub Copilot custom instructions](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- [Codex CLI — AGENTS.md guide](https://developers.openai.com/codex/guides/agents-md/)
- [Gemini CLI docs](https://developers.google.com/gemini-code-assist/docs/gemini-cli)

### Skill Directories & Collections
- [skills.sh](https://skills.sh/) — open directory of agent skills
- [Anthropic official skills](https://github.com/anthropics/skills)
- [awesome-agent-skills](https://github.com/skillmatic-ai/awesome-agent-skills) — curated list

---

## Contributing

Found an outdated API, broken command, or incorrect pattern? This skill is designed to be **self-correcting** — open a PR or issue and it benefits every future session.

---

## License

MIT

---

Built with the [Agent Skills open standard](https://agentskills.io)
