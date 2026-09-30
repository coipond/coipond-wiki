# Getting Started

This page takes you from an empty machine to your first running workflow: what to install, how to boot CoiPond for development, and a step-by-step first workflow.

## Prerequisites

CoiPond shells out to the **`coi`** binary for every container operation (it never talks to `incus` directly), so you need Incus/LXD and coi installed:

```shell
sudo snap install lxd && sudo lxd init   # Incus/LXD
cargo install coi                        # coi (or grab a release binary)
```

Prebuilt binaries: [coi releases](https://github.com/mensfeld/code-on-incus/releases/latest). CoiPond runs `coi` from `PATH`; override with `COI_BINARY_PATH`.

> **Minimum coi version is `v0.12.0`:** or a `master` build that includes [#745](https://github.com/mensfeld/code-on-incus/pull/745). CoiPond relies on coi applying a profile's `[tool.*]` config - e.g. `[tool.claude] model` - as container-level env that our `coi container exec` inherits; on older coi it silently falls back to the tool default. The System Status page checks this and flags an out-of-date binary.

You also need **PostgreSQL 18**. For development, `docker compose up -d` starts one (database only; the app itself runs via `bin/dev`).

## Quick Start

```shell
bin/setup --skip-server                                                 # deps + DB
bin/rails runner "Containers::Images::Build.call(image_type: :system)"  # build the base image
bin/dev                                                                 # web + watchers + worker + terminal daemon
```

`bin/dev` runs the web server, the JS/CSS watchers, the Solid Queue worker, and the terminal broadcast daemon (see `Procfile.dev`).

Then open <http://localhost:3000>, create a workflow in the editor, and trigger it. Copy `.env.example` to `.env` and fill in the required values (`DATABASE_URL`, `RAILS_MASTER_KEY`, `ANTHROPIC_API_KEY`) first; the rest are optional. See [Configuration](Configuration) for the full list of environment variables.

## Your First Workflow

1. **Create a workflow.** From the UI, create a new workflow and open the editor. Give it a name and, optionally, a **system prompt** (standing instructions every AI node inherits).
2. **Add a trigger node.** Drop a **trigger** node - this is the entry point. For your first run, leave it as a manual trigger.
3. **Add an AI node.** Drop an **ai** node and write its prompt, for example: *"In /workspace, add a CHANGELOG entry for the latest commit and stop."* Wire the trigger's success edge into it.
4. **Pick a coi profile.** The node's coi profile decides isolation, including network access (its `[network]` block). For anything touching git/GitHub and the model API, use a profile with an **allowlist**; the seeded default is **open**, fine while experimenting.
5. **Run it.** Hit **Run**. Watch the live terminal stream as the agent works. When it finishes, the workflow advances along the node's edges (or ends).
6. **Iterate.** Add a **shell** node before the AI node to set things up (checkout, install), an **approval** node after it to gate merges, or a failure edge to a notification step.

For what each node does in depth, see [Node Types](Node-Types).

> **Prompt for a clean finish:** start with a tiny, unambiguous prompt that ends in "and stop" so the first run finishes cleanly, then grow from there.
