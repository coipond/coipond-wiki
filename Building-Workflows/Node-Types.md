# Node Types

A node is one step in a workflow, and its type decides what the step does and how it reports success. CoiPond ships seven node types, registered in `Workflows::NodeTypes::ALL`.

## The Seven Node Types

- **ai** - runs a coding agent (Claude Code or Aider) headless on the node's `prompt`, autonomously. This is the workhorse.
- **shell** - runs `prompt` as a script in the same container and workspace. Use it for setup, builds, and test runs; its exit code drives success and failure routing.
- **interactive** - launches an agent session meant for a human to drive from the live terminal. It stays open for you rather than running to completion on its own.
- **approval** - a gate that pauses the workflow until someone approves in the UI. Put it before anything irreversible, such as a merge or a deploy. It shows `approval_message` while it waits.
- **sleep** - a timed pause for backoff or for spacing out steps. It pauses for `duration_seconds` (default 60).
- **trigger** - the entry point that carries the trigger configuration (manual, cron, or webhook). Set `is_entry_point: true`. See [Triggers](Triggers.md) for the full trigger model.
- **workflow_call** - invokes another workflow as a sub-step, so you can compose reusable pipelines out of smaller workflows.

Every non-terminal node has **success** and **failure** edges; wire them to route the run. For how `task_result` picks the edge, loops, and fan-out, see [Workflows](Workflows.md).

!!! note "Adding a type is self-contained"
    drop a strategy module under `app/concepts/workflows/node_types/` and register it in `Workflows::NodeTypes::ALL`; nothing else needs to know.

## How Each Type Decides Success

Each type reports its outcome differently. An `ai` node writes a `task_done` marker file; a `shell` node uses its exit code; an `interactive` node waits for you to type a verdict. The table below summarizes the signal per type.

<table>
<thead>
<tr><th>Type</th><th>What It Does</th><th>How It Decides Success</th></tr>
</thead>
<tbody>
<tr><td><code>trigger</code></td><td>Entry point. Set <code>is_entry_point: true</code>.</td><td>n/a</td></tr>
<tr><td><code>ai</code></td><td>Runs the agent headless on <code>prompt</code>.</td><td>The agent writes a <code>task_done</code> marker.</td></tr>
<tr><td><code>shell</code></td><td>Runs <code>prompt</code> as a script in the container.</td><td>Exit code. 0 is success.</td></tr>
<tr><td><code>interactive</code></td><td>Live terminal you drive yourself.</td><td>You type <code>//success</code> or <code>//failure</code>.</td></tr>
<tr><td><code>approval</code></td><td>Blocks on a human. Shows <code>approval_message</code>.</td><td>Approve or reject.</td></tr>
<tr><td><code>sleep</code></td><td>Pauses <code>duration_seconds</code> (default 60).</td><td>Always continues.</td></tr>
<tr><td><code>workflow_call</code></td><td>Invokes <code>target_workflow</code>.</td><td>Fire-and-forget, or wait on the batch.</td></tr>
</tbody>
</table>

!!! note "Shell scripts live in the `prompt` column"
    a `shell` node reads its script from the same `prompt` field an `ai` node uses for its task text.

## Common Node Fields

These fields apply across node types and are set per node, so a credentialed step and a sandboxed step can live in the same graph.

- **`coi_profile` / `incus_image`** - set per node. This is how a credentialed step and a sandboxed step live in the same graph.
- **`timeout_minutes`** - set per node. This is the enforcement point that kills a long-running session.
- **`run_once`** - the node fires only on its first successful run, recorded in `run_once_completed_at`. Useful for setup steps inside a loop.
- **`continue_prompt`** - pushed into the container as a second prompt file, for persistent nodes.
- **`nudge_on_api_error`** - opt-in auto-recovery. `Sessions::ApiErrorNudger` retries the session on an API error instead of failing it. Off by default so real failures elsewhere stay loud.
- **`config`** - a jsonb catch-all. Currently holds `await_count`.
