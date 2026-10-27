# Why CoiPond

CoiPond turns coding agents into infrastructure: a fleet of sessions that start from events, run isolated and unattended, hand off to each other, and pull in a human only when one is actually needed.

Coding agents like Claude Code and Aider are powerful, but on their own they are one terminal, one task, one human watching. CoiPond makes them a fleet instead.

Concretely, it lets you:

- Run an agent **on a schedule** (nightly dependency bumps, a recurring triage pass).
- Run an agent **in response to a webhook** (a GitHub PR opened, an issue labeled, any JSON event).
- **Chain** agent runs into pipelines, with success/failure routing between steps.
- Keep every run **isolated** in its own container so a misbehaving agent cannot touch your host or another run's workspace.
- **Drop into a live terminal** for any session and drive the agent by hand when you want to.
- Get an **answer prompt** in the UI when an agent asks a question mid-run, instead of a stuck session nobody notices.

Everything runs on your own hardware. There is no SaaS control plane and no data leaving your machines except the agent's own API calls.

## Use Cases

A few concrete things people build with CoiPond, and how they map to workflow nodes:

- **Nightly dependency updates.** A cron trigger fires an **ai** node ("update dependencies, run the test suite, open a PR if green"). Give its node a coi profile with an allowlist `[network]` (GitHub + your package registry + the model API), and mark the profile persistent so the workspace stays warm. Route failure to a second node that files an issue.
- **PR review bot.** A GitHub webhook (`pull_request.opened`) hits a trigger node; an **ai** node reviews the diff and posts comments. Because the payload is available to the run, the prompt can reference the PR number and branch.
- **Issue triage.** A webhook on `issues.labeled` runs an **ai** node that reproduces, diagnoses, and either proposes a fix branch or asks a clarifying question - which surfaces in the UI as a **blocked** session for a human to answer.
- **Scheduled refactor / cleanup passes.** A cron **ai** node walks a codebase applying a standing instruction (dead-code removal, comment hygiene, test backfill), gated by an **approval** node before anything merges.
- **Multi-step pipelines.** Chain a **shell** node (build/setup) into an **ai** node (implement) into another **shell** node (run tests) into an **approval** gate, with failure edges routing to a cleanup or notification step.
- **Interactive pairing at scale.** Use **interactive** nodes to spin up ready-to-drive agent terminals on demand, each in its own isolated container, and jump into whichever one you need from the browser.

## Features

- **Trigger-based automation** - cron schedules, webhooks (generic JSON + per-trigger token URLs), workflow chaining, and manual triggers.
- **Multi-step workflows** - pipelines of node types (AI task, shell, interactive, approval, sleep, sub-workflow call) with success/failure routing between steps.
- **Visual workflow editor** - a React canvas for wiring nodes together; workflows are stored as portable [CNCF Serverless Workflow](https://serverlessworkflow.io/) v1.0 documents.
- **Parallel execution** - up to 10 executions per workflow at once, each in its own isolated container "lane" (slot); triggers over the limit queue until a slot frees.
- **Autonomous *and* interactive sessions** - run an agent headless on a prompt, or drop into a live terminal and drive it yourself.
- **Isolated by default** - every session runs in its own Incus container via [coi](https://github.com/coipond/coi); what gets mounted and how the network is locked down is decided per session by a **coi profile**.
- **Human-in-the-loop** - approval gates and blocked-session handling when an agent asks a question mid-run.
- **Multi-machine ready** - scale from one box to a fleet of worker nodes sharing one database, with no change to how you author workflows.
- **Live terminals and visibility** - real-time xterm.js terminal streaming over SSE, admin status and per-node resource views, and centralized error tracking.

For the mechanics behind all of this, see [How It Works](How-It-Works.md), and to set it up see [Getting Started](index.md).
