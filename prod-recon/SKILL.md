---
name: prod-recon
description: >
  Read-only reconnaissance of a running deployment (production, stage or the local stack) for
  any project: find the way in on your own, confirm which commit runs there, read the logs and
  the database for the right time window, report a verdict with evidence, and stop without
  fixing anything. Always use it when the owner asks to look at a server without changing it,
  in any wording: "read-only", "recon and report", "take a look at prod", "check the server and
  report", "look at the logs on stage", "is the agent alive right now", "why did the bot stop
  responding", "were there any errors last week", "what's the state of the server", "which
  version is on prod". The owner may phrase this in another language, often as a single word
  meaning "read-only"; match on meaning.
---

# Read-only reconnaissance

This is the most frequent and the most expensive request the owner makes. Every time the agent
starts by asking which server, or explains the setup again, or, worst of all, finds the cause
and fixes it in the same turn without a word. The owner's exact reaction to the last one: "why
are you making changes without agreeing them when you haven't even reported what the problem
is". This skill exists to make the request cheap: find the way in yourself, read what matters,
report, stop.

## The contract

Read-only means the state of the server, the database, the repository and any project file is
identical before and after your work.

Allowed: `ssh`, and `sshpass` when the owner gave a login and password; `docker ps`,
`docker inspect`, `docker logs`, `docker compose ps`; `SELECT` queries; reading files with `cat`,
`grep`, `sed -n`, `tail`; `df`, `free`, `uptime`; a GET request to the public domain from your
machine.

Not allowed: restarting or recreating anything; `docker compose up/down/restart`; any
`docker ... prune`; `INSERT`, `UPDATE`, `DELETE`, `ALTER`; editing any file, local or remote,
including the project repository; cancelling or re-running CI jobs; copying keys or printing
secrets, tokens or passwords (when you decode a token to read its expiry, print only the
expiry).

A found cause does not grant the right to fix it. "Found it, fixed it" inside one turn is
forbidden even when the fix is one line. Reconnaissance ends with a report and a one-line
proposal; the fix is a separate request from the owner. If you catch yourself typing an editing
command, stop and report instead.

## Before the first command, say the plan

One or two sentences, out loud: what you will look at, in what order, what you will not touch,
and what the goal is. The wording the owner approved:

"I'll check stage strictly read-only: first the runbook, then the containers and the logs,
without restarting any service. Goal: link the failure to the release that went out today."

Then work. Do not ask which server, which key or where the stack lives; that is the next
section's job.

## Find the way in yourself

Look in this order and stop at the first source that answers:

1. **The project's deployment docs**: a runbook or deploy guide in `docs/`, `README`, the
   agent instructions file. They usually name the environments, hosts, stack paths, compose
   project names and the commands that are forbidden on that server.
2. **Agent memory** for this project: earlier sessions often recorded the alias, the access
   form and the traps.
3. **`~/.ssh/config`**: `Host` aliases carry the project and the environment in their names,
   with `HostName`, `User` and `IdentityFile` next to them. Use the alias, never the raw address
   with a key path.
4. **Deployment files in the repository**: compose files, `deploy/` and `.deploy/`, CI
   workflows. They give container and compose project names, the env file location, the
   release marker.

For the local stack, the source is the project's own dev compose and instructions.

When the owner put a login and password straight into the request, that is the way in: connect
with `sshpass` so the session is non-interactive. Hand the password over through the `SSHPASS`
environment variable and `sshpass -e`, not as a `-p` argument, so it does not land in the process
list or the shell history; do not save it anywhere and do not repeat it in the report. If
`sshpass` is not installed, install it with the system package manager (`sudo apt-get install -y
sshpass` or the equivalent). That installation is the one change to your own machine the
contract allows.

A host whose name merely resembles the project is not a match. If two candidates remain, or
none, say exactly what you found and what is missing, and ask one precise question. Ask only
then. Never pick a server by guess: a wrong read is harmless, but it produces a confident wrong
report.

## Which environment the owner means

Take it from the owner's words: production, stage or test, local. Cross-check with the
environment table in the deployment docs, the host alias and the domain, so that "stage" does
not land on a host that only used to be stage.

If the environment is not named and does not follow from the conversation, say so and ask.
Do not default to production, and do not check "all of them just in case": that multiplies the
time and blurs the report.

## The first fact: which commit runs there

Before reading a single log line, establish what code is actually running on the server the
owner pointed at. Otherwise every conclusion may be about a version that is not there.

Where to look, in order of reliability: the release marker the deployment writes (for example
a `current-release` file in the deploy directory, with SHA, method and time); the image tag of
the running container (`docker inspect --format '{{.Config.Image}}'`); `git rev-parse HEAD` if
the server holds a checkout; image labels if the build sets them.

Then place it in the repository: `git log -1 <sha>` for the subject and date,
`git branch -a --contains <sha>` for the branches, `git rev-list --count <sha>..origin/<branch>`
for how far behind. Report it as one line: "production runs `<sha>` (<subject>, <date>),
N commits behind `<branch>`". For the local stack: branch, HEAD, dirty tree, and whether the
running processes use a stale build rather than the current sources.

If no source yields a version, write "version could not be determined" and go on. Never write
"probably the latest".

## What to read, and for how long

Pick the time window from the question and state it in the report. An incident happening now:
the last one or two hours. "Over the last week": seven days. Nothing said: the last 24 hours.

Read in this order; each step narrows the next one.

1. **Containers.** State, `StartedAt`, `RestartCount`. A restart is an event to correlate with
   errors, not evidence of health.
2. **Application logs**, `docker logs --since <window> <container>`. Count first, read second:
   `grep -c` for error codes and classes, `sort | uniq -c | sort -rn | head` for the top ones,
   then the first occurrence, then the lines around it. Separate recurring noise from what is
   new in the window. Line up timestamps with restarts and releases.
3. **The database**, `SELECT` only. Statuses of the recent domain entities (runs, jobs,
   messages) in the window: counts by status, the time of the last successful completion, items
   stuck in an in-progress state longer than the application's own timeout. Take column names
   from `information_schema.columns`, not from the ORM models: table and column names differ.
   Pass SQL through stdin (a heredoc or a file), not as a `-c` argument: quotes travel through
   ssh and two shells and break.
4. **Gateways and providers** the application depends on: their logs for auth failures and
   timeouts, their own tables if they keep any. Status flags the application stores about them
   ("active", "connected") are claims; verify them by activity in the window.
5. **From the outside.** GET the public domain the users see, from your machine, not from the
   server: status code and response time for the health endpoint and the main page.

Bound every command: `timeout`, `ssh -o ConnectTimeout=10`, `--since`, `| head`. Never dump a
whole log into the context; count and filter first. Before a command that may take long, say
what is already established: silence longer than a few minutes reads as a hang.

## Which conclusions are allowed

- **"Works" only by a user-visible signal**: the bot replied at a given time, a run completed,
  the page answered 200 from outside, the version on the server matches the expected one. A
  `healthy` container, a live process and a green CI job prove that the process is up, not that
  the product works. This is the owner's most repeated complaint.
- **A red deployment job does not mean the release did not land.** Checks often run after the
  switch. Read the release marker on the server before concluding.
- **Stored status fields are claims.** An account marked active can hold an expired token; a job
  marked running can be long dead. Verify by activity: last request served, last row written,
  token expiry decoded from the credential itself.
- **A fast probe does not clear an intermittent failure.** With several accounts or instances
  behind a balancer, one good response can come from the healthy one. Look at the distribution
  over the window.
- **No errors in the window does not mean no problem.** Silence can mean nothing was processed.
  Check activity counts next to error counts.
- **Coincidence is not cause.** Errors that start after a restart or release "coincide with"
  it; say "caused by" only when the log says so.
- **Facts and hypotheses apart.** A fact carries a time and a source. A hypothesis is labelled
  as one.
- **Shared servers.** Another project's failure on the same host or the same CI runner is
  named and left alone, never touched, never fixed in passing.

## The report

Use this shape every time; the owner compares reports across days.

1. **Verdict, first line**: works or not, what exactly is broken, since when.
2. **Environment and commit**: which server, which SHA, how far behind which branch, the time
   window you read.
3. **Facts**, each with a timestamp and a source. One quoted log line and its interpretation
   right after it. No raw command output, no stack traces.
4. **Not checked**: what you could not verify and why. This line is mandatory even when it is
   "nothing".
5. **Proposal, one line**: what you would do next if the owner says so.

Then stop. Do not start the fix, do not open files for editing, do not offer "I can do it right
now". The next step is the owner's command.

## Working style

If the question is "is it alive", three commands and a GET may be enough. If the question is
"what happened last week", read wider, but still count before you read.
