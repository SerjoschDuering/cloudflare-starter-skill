# Cloudflare Starter Skill

**Deploy fullstack web apps on Cloudflare — guided by your AI coding agent.**

An open skill that teaches any AI coding agent (Claude Code, Cursor, Windsurf, Codex, Copilot, Gemini CLI, and more) how to build and deploy fullstack applications on Cloudflare's edge network. Free tier, no credit card, scales to zero.

Built for students and beginners. You make the decisions — the AI executes.

---

## What Is a "Skill"?

A **skill** is a set of markdown files that give AI coding agents specialized knowledge. Instead of the AI relying on its training data (which may be outdated), it reads these files at runtime for up-to-date, executable instructions.

Think of it like giving your AI a cheat sheet — except it's 10 reference docs covering everything from project setup to CI/CD deployment.

This skill follows the [Agent Skills open standard](https://agentskills.io), which is supported by all major AI coding tools.

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

Everything runs at the edge. No servers to manage, no cold starts to worry about.

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

## Service Decision Matrix

| Need | Service | Free Tier |
|------|---------|-----------|
| API / serverless functions | **Workers** | 100K req/day |
| Frontend SPA or SSR | **Workers + Static Assets** | Unlimited bandwidth |
| Relational database (SQL) | **D1** (SQLite) | 5M reads/day, 5GB |
| File/image/video storage | **R2** (S3-compatible) | 10GB, zero egress |
| Sessions, config, cache | **KV** (key-value) | 100K reads/day |
| User authentication | **Better Auth** (OSS) | Free |
| Background jobs | **Queues** | 10K ops/day |
| Existing Postgres/MySQL | **Hyperdrive** (proxy) | 100K queries/day |
| AI inference | **Workers AI** | 10K neurons/day |
| Bot/spam protection | **Turnstile** | 1M req/month |

---

## Setup by Tool

### Claude Code

Add the skill to your project:

```bash
claude mcp add-skill SerjoschDuering/cloudflare-starter-skill
```

Or clone it locally and reference the path:

```bash
git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git
# Then in your project's .claude/settings.json, add the skill path
```

### Cursor / Windsurf

Clone into your project and the tool will automatically pick up `SKILL.md`:

```bash
git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .skills/cloudflare
```

Or add as a git submodule:

```bash
git submodule add https://github.com/SerjoschDuering/cloudflare-starter-skill.git .skills/cloudflare
```

### Codex / Gemini CLI / Cline / Roo Code

These tools look for `AGENTS.md`. Clone the repo into your project — `AGENTS.md` is included as a mirror of `SKILL.md`:

```bash
git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .skills/cloudflare
```

### Any Other Tool

The skill is plain markdown. Clone it anywhere and point your tool at the files:

```bash
git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git
```

---

## Quick Start

Once the skill is loaded, just tell your AI agent what you want to build:

> "I want to build a todo app on Cloudflare"

> "Help me deploy a fullstack React app with a database"

> "I have no idea what to build — give me project ideas"

The agent will read the appropriate reference files and guide you through every step — from account setup to production deployment.

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

## Contributing

Found an outdated API, broken command, or incorrect pattern? This skill is designed to be **self-correcting** — open a PR or issue and it benefits every future session.

---

## License

MIT

---

Built with the [Agent Skills open standard](https://agentskills.io)
