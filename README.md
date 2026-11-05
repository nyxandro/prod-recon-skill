<div align="center">

# Prod Recon

**A Claude Code skill for read-only checks of production. Your AI agent finds the server, reads logs and the database, reports with evidence and touches nothing.**

[![Release](https://img.shields.io/github/v/release/nyxandro/prod-recon-skill)](https://github.com/nyxandro/prod-recon-skill/releases)
[![License](https://img.shields.io/github/license/nyxandro/prod-recon-skill)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-SKILL.md-6f42c1)](https://agentskills.io/specification)
[![Claude Code plugin](https://img.shields.io/badge/Claude%20Code-plugin-d97757)](#install)

**English** · [Русский](README.ru.md)

</div>

Prod Recon is a skill for read-only reconnaissance of a running deployment, whether production,
stage or the local stack, in Claude Code, Codex, OpenCode, Cursor and other agents that read Agent
Skills. The agent finds the way in on its own, confirms which commit is actually running, reads
the logs and the database for the right time window, reports a verdict with evidence and stops
without fixing anything.

## Why

"Take a look at prod, read-only" is the request owners make most often, and it goes wrong the same
way every time.

- The agent asks which server, though the answer is in the repo.
- It explains the setup back to you again.
- It finds the cause and quietly fixes it in the same turn, before telling you what was wrong.

This skill makes the request cheap and the outcome predictable.

## Install

| Agent | Command |
|---|---|
| Claude Code, as a plugin | `/plugin install prod-recon --marketplace nyxandro/skills` |
| Claude Code, Codex, OpenCode, Cursor and others | `npx skills add nyxandro/prod-recon-skill` |
| Any agent, by hand | Download the [latest `.skill` archive](https://github.com/nyxandro/prod-recon-skill/releases/latest/download/prod-recon.skill) and unpack it into the agent's skills folder |

Pick one. If you install it two ways, Claude Code loads the skill twice.

<details>
<summary>More install options</summary>

<br>

- The one-line plugin install needs Claude Code 2.1.275 or later. On an older version, add the
  catalog first with `/plugin marketplace add nyxandro/skills` and then run
  `/plugin install prod-recon@nyxandro`.
- The catalog pins no version, so every commit here reaches plugin users as an update. Run
  `claude plugin marketplace update nyxandro` and `claude plugin update prod-recon@nyxandro`, or
  turn on auto-update under `/plugin` → Marketplaces.
- With `npx skills`, add `-g` to install the skill for every project. To install a given version,
  add its tag, as in `nyxandro/prod-recon-skill@v1.0.0`.
- You can also clone this repository and link the `prod-recon/` folder into your agent's skills
  folder. The skill is a single `SKILL.md`, with no scripts and no build step.

</details>

## Usage

Often one word is enough. These requests all start the skill.

- *Read-only.*
- *Take a look at prod and report.*
- *Look at the logs on stage.*
- *Why did the bot stop responding?*
- *Were there any errors last week?*
- *Which version is on prod?*

## How it works

1. The agent states its plan in a sentence or two before the first command. For example, "I'll
   check stage strictly read-only. First the runbook, then the containers and the logs, without
   restarting anything. The goal is to link the failure to today's release."
2. It finds the way in without asking. The project's deployment docs, its memory of earlier
   sessions, `~/.ssh/config` aliases and the deployment files in the repo. A login and password in
   your request are used through `sshpass`. A host that only resembles the project doesn't count,
   and if you didn't name the environment, it asks instead of picking production.
3. It establishes which commit runs on that server and how far behind which branch. "Probably the
   latest" is never an answer.
4. It reads for a time window taken from your question. Containers and their restarts, application
   logs counted before they're read, the database with `SELECT` only, the services the app depends
   on, and the public domain from outside.
5. It draws only the conclusions the evidence allows. "Works" means a signal a user would see, not
   a healthy container or a green CI job.
6. It reports and stops. The fix is a separate request.

## The read-only contract

| Allowed | Not allowed |
|---|---|
| `ssh`, and `sshpass` when you gave a login and password | Restarting or recreating anything |
| `docker ps`, `docker inspect`, `docker logs`, `docker compose ps` | `docker compose up`, `down` or `restart`, any `prune` |
| `SELECT` queries | `INSERT`, `UPDATE`, `DELETE`, `ALTER` |
| Reading files, `df`, `free`, `uptime` | Editing any file, local or remote |
| A GET to the public domain from outside | Cancelling or re-running CI jobs, printing secrets |

A found cause doesn't give the right to fix it, even when the fix is one line.

## Example

A report from a made-up incident. The shape is the same every time, so you can compare days.

```text
Verdict. The bot has not replied since 14:05. The LLM gateway rejects every call.

Environment. Production, alias acme-prod. Runs 3f9c2a1 "Switch to the new model",
deployed today, 0 commits behind main. Window 12:00 to 16:00.

Facts
  14:02  The release marker shows 3f9c2a1 went live.
  14:05  First of 412 bot log lines "401 invalid api key". Every LLM call since then fails.
  15:58  GET of the health page from outside returns 200 in 0.3 s, so the web part is fine.

Not checked. The gateway's own dashboard, there is no access to it.

Proposal. Put the new key into the production env and restart the bot, on your command.
```

## Versions

The current version sits in `metadata.version` of `prod-recon/SKILL.md`, so any installed copy
tells you which one it is. Every version has a git tag and a
[release](https://github.com/nyxandro/prod-recon-skill/releases) with its own archive.
[CHANGELOG.md](CHANGELOG.md) lists what changed.

<details>
<summary>Repository layout</summary>

<br>

```text
prod-recon/SKILL.md       the skill
dist/prod-recon.skill     the skill folder as a downloadable archive
CHANGELOG.md              what changed in each version
README.md, README.ru.md   this page in English and in Russian
```

</details>

## License

[MIT](LICENSE)
