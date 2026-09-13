# Prod Recon Skill

[![Download Skill](https://img.shields.io/badge/Download-prod--recon.skill-blue?style=for-the-badge)](https://github.com/nyxandro/prod-recon-skill/raw/main/dist/prod-recon.skill)

A skill for Claude Code and OpenCode: read-only reconnaissance of a running deployment for any
project. Production, stage or the local stack. The agent finds the way in on its own, confirms
which commit is actually running there, reads the logs and the database for the right time
window, reports a verdict with evidence, and stops without fixing anything.

"Take a look at prod, read-only" is the most frequent request an owner makes to an agent, and
the one that goes wrong the same way every time: the agent asks which server, or re-explains the
setup, or finds the cause and fixes it in the same turn without a word. This skill makes the
request cheap and the outcome predictable.

## What the agent does

1. Says the plan in one or two sentences: what it will look at, in what order, what it will not
   touch, and why.
2. Finds the way in without asking: the project's deployment docs, its own memory,
   `~/.ssh/config` aliases, the deployment files in the repository. A login and password given
   in the request are used through `sshpass`. A host that merely resembles the project is not a
   match; if the environment is not named, it asks instead of defaulting to production.
3. Establishes the first fact: which commit runs on that server, and how far behind which
   branch. "Probably the latest" is not an answer.
4. Reads in order, with a time window taken from the question: containers and their restarts,
   application logs counted before they are read, the database with `SELECT` only, the gateways
   the application depends on, and the public domain from the outside.
5. Draws only the conclusions the evidence allows. "Works" is proven by a user-visible signal,
   not by a `healthy` container or a green CI job. A red deployment job does not mean the
   release did not land. Stored status fields are claims until activity confirms them.
6. Reports in a fixed shape: verdict first, environment and commit, facts with timestamps and
   one interpreted log line each, what was not checked and why, one line of proposal.
7. Stops. The fix is a separate request.

## The read-only contract

Allowed: `ssh` and `sshpass`, `docker ps`, `docker inspect`, `docker logs`, `docker compose ps`,
`SELECT` queries, reading files, `df`, `free`, `uptime`, a GET to the public domain.

Not allowed: restarting or recreating anything, `docker compose up/down/restart`, any
`docker ... prune`, writes to the database, editing any file local or remote, cancelling or
re-running CI jobs, copying keys or printing secrets. A found cause does not grant the right to
fix it, even when the fix is one line.

## Install

Download `dist/prod-recon.skill` (the badge above) and unpack it into the skills folder of your
agent, or clone this repository and link the `prod-recon/` folder there. The skill is a single
`SKILL.md`; no scripts, no build step.

## Repository layout

```text
prod-recon/SKILL.md     the skill
dist/prod-recon.skill   downloadable archive of the skill folder
```

## License

MIT.
