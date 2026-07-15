<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://flusterduck.com/logo.png">
    <img src="https://flusterduck.com/logo-orange.png" alt="Flusterduck" width="120">
  </picture>
</p>

<h1 align="center">flusterduck <sub>agent skills</sub></h1>

<p align="center">Skills that teach an AI agent to do real product work with <a href="https://flusterduck.com">Flusterduck</a>, automatic issue tracking for UX friction.</p>

---

Flusterduck watches how people actually use your site (rage clicks, dead clicks, form abandonment, tap misses) and clusters that friction into UX issues with evidence and recommended fixes. These four skills give any agent that reads a skills directory the procedures to work with that data well: what to pull, in what order, how to rank, and when to say "not enough evidence" instead of guessing.

## Install

With the skills CLI:

```bash
npx skills add flusterduck/skills
```

Just one skill:

```bash
npx skills add flusterduck/skills --skill flusterduck-triage
```

Or manually, into Claude Code's skills directory:

```bash
mkdir -p ~/.claude/skills/flusterduck-triage
curl -o ~/.claude/skills/flusterduck-triage/SKILL.md https://flusterduck.com/skills/flusterduck-triage.md
```

Every skill file is also listed with a sha256 hash at [`flusterduck.com/.well-known/agent-skills/index.json`](https://flusterduck.com/.well-known/agent-skills/index.json) so you can verify what you downloaded.

## The skills

| Skill | Use it when |
|---|---|
| [`flusterduck-setup`](flusterduck-setup/SKILL.md) | Installing Flusterduck: the SDK or a framework wrapper on the site, and the MCP server in your agent. |
| [`flusterduck-triage`](flusterduck-triage/SKILL.md) | "What should I fix?" Turns open UX issues into a ranked plan split into Fix this and Watching. |
| [`flusterduck-investigate-page`](flusterduck-investigate-page/SKILL.md) | One route is underperforming and you want to know where users get stuck and why. |
| [`flusterduck-verify-fix`](flusterduck-verify-fix/SKILL.md) | You shipped a fix. Compares confusion before and after the deploy: HELD, REGRESSED, or PENDING. |

## Prerequisites

The setup skill works with nothing but a Flusterduck account. The three data-reading skills need live data access, either of:

- the **Flusterduck MCP server** (`@flusterduck/mcp-server`) connected with an `fd_mcp_` key, or
- an `fd_sec_` secret key for the REST API at `api.flusterduck.com/v1`.

The setup skill walks an agent through both. Flusterduck is a paid product with a 3-day free trial.

## About this repo

This repository is a read-only mirror, maintained automatically from the Flusterduck product repo. Issues and pull requests here are not monitored; send feedback through [docs.flusterduck.com](https://docs.flusterduck.com) or the in-product feedback form.

---

<p align="center">
  <a href="https://docs.flusterduck.com">Documentation</a> ·
  <a href="https://docs.flusterduck.com/rest-api">REST API</a> ·
  <a href="https://docs.flusterduck.com/mcp">MCP server</a>
</p>
