# Workflows

A workflow is a directed graph of nodes. Each node runs as a session inside its own Incus container. When a node finishes it routes to `on_success_node` or `on_failure_node`, and the graph keeps walking until it hits a node with no outgoing edge. This page is the deep reference for how workflows behave.

For the container and profile side see [How CoiPond Uses Coi](../Internals/How-CoiPond-Uses-Coi.md); for network policy see [Network Profiles](../Configuration/Network-Profiles.md).

## Object Model

<table>
<thead>
<tr><th>Model</th><th>Table</th><th>Role</th></tr>
</thead>
<tbody>
<tr><td><code>Workflows::Workflow</code></td><td><code>workflows_workflows</code></td><td>The graph. Enabled/draft/hidden, concurrency cap, PaperTrail versioned.</td></tr>
<tr><td><code>Workflows::Trigger</code></td><td><code>workflows_triggers</code></td><td>How a run starts: <code>webhook</code>, <code>scheduled</code>, or <code>manual</code>.</td></tr>
<tr><td><code>Workflows::Node</code></td><td><code>workflows_nodes</code></td><td>One step, plus its outgoing edges.</td></tr>
<tr><td><code>Workflows::Execution</code></td><td><code>workflows_executions</code></td><td>One run of the whole graph. Snapshots config at start.</td></tr>
<tr><td><code>Workflows::Session</code></td><td><code>workflows_sessions</code></td><td>One node actually executing in a container.</td></tr>
<tr><td><code>Workflows::AwaitGroup</code></td><td><code>workflows_await_groups</code></td><td>Join point when a node waits on child workflows.</td></tr>
</tbody>
</table>

A workflow can have several triggers. Each fires an `Execution`, which spawns a `Session` per node as the graph advances. One node visited twice (via a loop) means two sessions.

## Triggers

`trigger_mode` is one of the following. See [Triggers](Triggers.md) for the full trigger reference.

- **`scheduled`** - cron. `Triggers::CheckScheduled` picks up rows where `next_check_at <= now`.
- **`webhook`** - every trigger gets a `webhook_token` on create, giving it a private URL at `POST /api/webhooks/trigger/:token`. There is also a generic receiver at `POST /api/webhooks/ingest`.
- **`manual`** - the button in the UI. Takes no parameters: `TriggerManually` sets `trigger_context` to `{triggered_by: "manual"}` and nothing else. If a run needs an input (which repo, which doc), either bake it into the node prompts, use a webhook trigger and read the payload from `TRIGGER.md`, or start with an `interactive` node and type it.

Workflows also chain: `Workflow#trigger_workflow` links one to another, and a `workflow_call` node invokes another workflow directly (see [Fan-out and Fan-in](#fan-out-and-fan-in)).

## Node Types

Seven types, registered in `Workflows::NodeTypes::ALL`. Adding one means dropping a strategy module under `app/concepts/workflows/node_types/` and registering it there; nothing else needs to know. See [Node Types](Node-Types.md) for the per-type reference.

<table>
<thead>
<tr><th>Type</th><th>What It Does</th><th>How It Decides Success</th></tr>
</thead>
<tbody>
<tr><td><code>trigger</code></td><td>Entry point. Set <code>is_entry_point: true</code>.</td><td>n/a</td></tr>
<tr><td><code>ai</code></td><td>Runs the agent headless on <code>prompt</code>.</td><td>The agent writes a <code>task_done</code> marker (below).</td></tr>
<tr><td><code>shell</code></td><td>Runs <code>prompt</code> as a script in the container.</td><td>Exit code. 0 is success.</td></tr>
<tr><td><code>interactive</code></td><td>Live terminal you drive yourself.</td><td>You type <code>//success</code> or <code>//failure</code>.</td></tr>
<tr><td><code>approval</code></td><td>Blocks on a human. Shows <code>approval_message</code>.</td><td>Approve or reject.</td></tr>
<tr><td><code>sleep</code></td><td>Pauses <code>duration_seconds</code> (default 60).</td><td>Always continues.</td></tr>
<tr><td><code>workflow_call</code></td><td>Invokes <code>target_workflow</code>.</td><td>Fire-and-forget, or wait on the batch.</td></tr>
</tbody>
</table>

`shell` nodes read their script from the `prompt` column, the same field the `ai` nodes use for their task text.

### Common Node Fields

- `coi_profile` / `incus_image` - per node. This is how a credentialed step and a sandboxed step live in the same graph.
- `timeout_minutes` - per node.
- `run_once` - the node fires only on its first successful run, recorded in `run_once_completed_at`. Useful for setup steps inside a loop.
- `continue_prompt` - pushed into the container as a second prompt file, for persistent nodes.
- `nudge_on_api_error` - opt-in auto-recovery. `Sessions::ApiErrorNudger` retries the session on an API error instead of failing it. Off by default so real failures elsewhere stay loud.
- `config` - jsonb catch-all. Currently holds `await_count`.

## Routing

Three rules, all in `Workflows::Sessions::Advance`:

1. **`task_result` picks the edge.** `success` takes `on_success_node`, `failure` takes `on_failure_node`.
2. **A crash is not a failure.** If the session ends up `errored` (container died, agent never reported an outcome), the pipeline stops. It does not take the failure edge. That edge is reserved for a deliberate "I could not do this" signal so an operator can inspect a genuine crash rather than watch it cascade into cleanup logic.
3. **No target means the path ends.** Not an error, just the end of that branch.

The next session inherits `workspace_path` and `branch_name` from the one that spawned it, and gets a `trigger_context` recording which node it came from.

### How task_result Actually Gets Set

This is the part worth internalizing, because it differs per node type.

**`ai` nodes** signal by writing a marker file. The `session_automated` system prompt is appended to every autonomous session and instructs the agent to do an atomic write:

```shell
cat > "$COIPOND_MARKER_DIR/task_done.tmp" <<EOF
{"result": "success", "at": "<ISO8601>", "run_id": "$COIPOND_RUN_ID"}
EOF
mv "$COIPOND_MARKER_DIR/task_done.tmp" "$COIPOND_MARKER_DIR/task_done"
```

`Sessions::MarkerWatcher` tails `$COIPOND_MARKER_DIR` with inotify and `MarkerHandler` turns the event into a state change. The `.tmp` plus `mv` is required: the watcher assumes atomic writes and will otherwise read a half-written file.

There is no alternative signal path. Printing text to stdout does nothing. A Stop hook fires at the end of every response turn and exits 2 if the marker is missing, which pushes the agent to either keep working or declare an outcome.

**`shell` nodes**: `ShellMonitor` maps exit code 0 to `success`, anything else to `failure`.

**`interactive` nodes**: the operator types `//success`, `//failure`, or `//exit` (which counts as success). `InteractiveExit` intercepts the keystrokes before they reach the pane and writes the marker.

**Everything else** falls back to `completed` meaning success.

### Branching Without a Condition Node

There is no `condition` node type. You branch with an `ai` node whose verdict is its `task_result`, then wire both edges.

The prompt has to say so explicitly. Do this:

> Check whether the test is flaky (passes on some runs, fails on others). If the success rate is between 10% and 90%, write `task_done` with `{"result": "success"}`. Otherwise write `{"result": "failure"}`. The result is a verdict about the test, not about whether your analysis worked.

That last sentence matters. The default reading of "success" is "my task went fine," which is exactly backwards for a decision node, and an agent left to its own judgment will report success for a well-executed analysis that concluded "not flaky."

!!! warning "The seed's print convention is outdated"
    `db/seeds/flaky_test_workflow_nodes.rb` predates this. Its decision node asks the agent to print `SUCCESS` or `FAILURE` as the last line of output, which nothing reads. The graph shape in that seed is a good reference; the prompt convention is not.

### Loops

A loop is a self-edge on the failure path. `Node#can_loop?` is literally `on_failure_node_id == id`. Retry-until-it-works is a node that fails back into itself.

A two-node cycle works the same way and is often what you actually want: a `shell` test node whose failure edge goes to a `fix` node, whose success edge goes back to the test node. Routing just follows edges, so cycles of any length run fine. (`can_loop?` only detects the one-node case, so it won't flag a longer cycle in the editor.)

Remember rule 2: only a reported `failure` loops. A crash exits the loop and halts the run, so a wedged container will not spin forever.

**Nothing bounds iteration count.** There is no working retry cap and no budget enforcement (see [Guardrails](#guardrails)). A test/fix cycle keeps going as long as each session reports a clean failure. If you need a bound, make it part of the task: have the loop node increment a counter file in the workspace and report `failure` once it passes your limit, so the run exits down a path you chose.

## Guardrails

Only one thing actually stops a session automatically:

- **`Node#timeout_minutes`** schedules a `Sessions::TimeoutJob` when the session spawns (`Sessions::Spawn:59-62`). This is real, and it is per node.

Things that look like guardrails but are not:

- **`Workflow#timeout_minutes`** (default `60`) is only rendered into `WORKFLOW.md` as text. It schedules nothing. Node-level timeouts are the enforcement point.

So set `timeout_minutes` on every long-running node; the workflow-level timeout is advisory. There is no budget cap and no automatic retry. If you need a bound, build it into the task (see [Loops](#loops)).

## Fan-out and Fan-in

`workflow_call` has two modes, and they are genuinely different code paths.

**Fire-and-forget** (`wait_for_completion: false`) spawns the target workflow and moves on to `on_success_node` immediately.

**Wait** (`wait_for_completion: true`) creates an `AwaitGroup` with `expected_count = await_count`, spawns that many copies of the target, and parks the calling session in `awaiting`. Each child increments `completed_count`; the parent wakes when they all finish.

`await_count` lives in `config["await_count"]`, defaults to 1, and is clamped to 1..10. That is your parallelism dial.

The group settles into one of:

<table>
<thead>
<tr><th>Status</th><th>Meaning</th></tr>
</thead>
<tbody>
<tr><td><code>completed</code></td><td>All children done, none failed.</td></tr>
<tr><td><code>completed_with_failures</code></td><td>All children done, at least one errored.</td></tr>
<tr><td><code>errored</code> / <code>cancelled</code></td><td>The group itself broke or was called off.</td></tr>
</tbody>
</table>

So you can distinguish "the batch finished" from "the batch finished and three blew up" before deciding where to route.

## Human in the Loop

Three distinct mechanisms:

- **`approval` node** -> session goes `pending_approval` and waits for an explicit approve/reject.
- **`blocked` session** -> an `ai` node hit something it genuinely cannot decide. This only works if the agent calls the `AskUserQuestion` tool. Printing a question as prose does not pause anything; the session finalizes on the next Stop. The `session_automated` prompt is emphatic about this.
- **`interactive` node** -> you are in the pane from the start, driving it, exiting with `//success` or `//failure`.

`Session::NEEDS_REVIEW_STATUSES` is `blocked` plus `pending_approval`, and backs the "needs attention" surfaces in the dashboard.

## Session Lifecycle

```text
pending -> queued -> running <-> blocked / awaiting / pending_approval
                       |
                       +-> sleeping (timed wait) -> running
                       +-> completed | errored | skipped   (terminal)
```

Terminal is `completed`, `errored`, `skipped` (`Session::TERMINAL_STATUSES`). Everything else is derived from the enum, so a newly added status counts as non-terminal until someone deliberately classifies it. The status groups in `Workflows::Session` are the single source of truth here; re-listing statuses inline is how `active_sessions` and `terminal?` drifted apart over `awaiting` and `skipped` once already.

Executions aggregate their sessions: any session reporting `failure` or ending `errored` makes the execution's `task_result` a failure.

## Workspace, Repos, and Context Files

Every session gets a host directory bind-mounted to `/workspace` in its container. Resolution order is in `Sessions::Spawn#setup_workspace`:

1. A node on a persistent coi profile gets a stable per-node dir (`persist-node-<id>`), so coi's slot allocator maps it to the same container every time.
2. A session with a pre-set `workspace_path` reuses it. This is how the graph stays coherent: `Advance` passes `workspace_path` and `branch_name` to the session it spawns, so every node in a run shares one directory.
3. Otherwise `Workspaces::Manager` mints a fresh per-session UUID dir.

**The workspace starts empty.** Nothing clones a repo for you. There is no `git_clone_url` anywhere in the schema. If a workflow operates on a repo, an early node has to clone it, either a `shell` node running `git clone` or the first `ai` node doing it as part of its task. `Workspaces::Manager#main_repo_path` finds the first git repo inside the workspace, so the convention is to clone into a subdirectory.

### Git Behaviour Is Prompt-Driven

There are no git-automation settings on a workflow. Branching, committing, and opening a PR happen only because a node's prompt tells the agent to do them. There are no `create_branch` / `auto_commit` / `create_pr` flags to toggle. If a step must produce a branch or a PR, say so in the node prompt, and put `GH_TOKEN` in that node's coi profile so it has the credential.

### Files Dropped Into the Workspace

- **`WORKFLOW.md`** - the workflow's structure and current execution state: every node with its prompt (truncated to 150 chars) and edges, which node is current, which steps have completed or failed. An agent can see where it sits in the graph.
- **`TRIGGER.md`** - how this run started: trigger type, schedule, and the full webhook payload when there is one. Only written when there's trigger context. This is the channel for per-run inputs. A webhook carrying `{"repo": "..."}` shows up here, and the file explicitly tells the agent to use it "instead of hardcoding values."
- **`AWAITED/`** - see below.

### Passing Data Between Workflows

A child workflow writes into a `RESULT/` directory in its workspace. When the await group settles, `Executions::CopyChildResults` copies each child's `RESULT/` to `AWAITED/<result_dir_name>/` in the parent's workspace, so the parent's continuation prompt can read what its children produced.

That's the only structured channel between a parent and its children. Within a single run, nodes just share the workspace.

## Secrets and Containers

Environment variables live in a single flat namespace (`/environment_variables`), one encrypted value per name. There is no per-workflow or per-node scoping.

A variable only reaches a session if that session's coi profile lists it in `forward_env`:

```toml
forward_env = ["GH_TOKEN", "MY_API_KEY"]
```

So access is decided by which profile a node picks, not by the workflow. Design your graph around that: the node that opens a PR wants a profile carrying `GH_TOKEN`, and the node that runs untrusted code wants one that carries nothing.

Nodes do not share a process. State moves between them through the workspace, git, or persistent storage (`persistent_storage_enabled`, mounted at `/storage`), never through variables.

### Keep Credentials Out of the Agent Node

Run the step that needs a credential (for example `git push`) in its own node, and give the `ai` node a profile that forwards no secrets. The agent then never holds the credential: it is not in the agent's session, so the agent cannot leak it, misuse it, or be talked into sending it somewhere.

For example, two profiles:

```toml
# agent profile, used by the ai node
# no forward_env, so no secret reaches this session
[network]
mode = "restricted"
```

```toml
# push profile, used by the shell node that pushes
forward_env = ["GH_TOKEN"]

[network]
mode = "allowlist"
allowed_domains = ["github.com", "api.github.com", "8.8.8.8", "1.1.1.1"]
```

Then wire `ai` (make the change, commit in the workspace) -> `shell` (push the branch) on the push profile. The `allowlist` limits where the token can go even from the push node; see [Network Profiles](../Configuration/Network-Profiles.md).

Both nodes share the workspace, so the push node acts on files the agent wrote. Keep it a fixed `shell` script with one narrow job, rather than a second `ai` node that follows instructions it finds in the workspace.

## Authoring

Three doors, all writing the same rows.

**1. The visual editor** at `/workflows/workflows/:id/nodes/editor`. Node canvas positions (`position_x`, `position_y`) and the workflow's viewport x/y/zoom are persisted, so layout survives.

**2. A definition document.** `Workflows::DefinitionSerializer` and `DefinitionParser` round-trip CNCF Serverless Workflow v1.0. Import is two-pass: nodes are created first, then name references like `on_success: "Analyze Code"` resolve to IDs. Each node type maps to a CNCF state:

<table>
<thead>
<tr><th>Node Type</th><th>CNCF State</th></tr>
</thead>
<tbody>
<tr><td><code>ai</code>, <code>shell</code>, <code>interactive</code></td><td><code>operation</code></td></tr>
<tr><td><code>trigger</code></td><td><code>event</code></td></tr>
<tr><td><code>approval</code></td><td><code>callback</code> (waits on <code>approval-response-&lt;id&gt;</code>)</td></tr>
<tr><td><code>sleep</code></td><td><code>sleep</code>, ISO-8601 duration</td></tr>
<tr><td><code>workflow_call</code></td><td><code>operation</code>, or <code>foreach</code> when waiting</td></tr>
</tbody>
</table>

Older documents may carry legacy function refs (`claude`, `claude-git`, `shell-git`). The strategies register those aliases, so old definitions still import.

**3. Ruby**, as the seeds do. Create the nodes, then wire the edges in a second pass:

```ruby
workflow = Workflows::Workflow.find_by(name: "Flaky Test Detector")

check = workflow.nodes.create!(
  name: "Is Test Flaky?",
  node_type: "ai",
  prompt: "...write task_done with result success if flaky, failure otherwise...",
  timeout_minutes: 3,
  coi_profile: CoiProfile.find_by(name: "claude-sonnet") || CoiProfile.default
)

check.update!(on_success_node: handle_flaky, on_failure_node: report_consistent)
```

## Worked Shapes

**Gate a risky change.** `trigger` -> `ai` (make the change) -> `shell` (run the suite) -> `approval` -> `ai` (open the PR). The suite's exit code routes, so a red build never reaches the human. Put `GH_TOKEN` in the profile on the last node only. See [Keep Credentials Out of the Agent Node](#keep-credentials-out-of-the-agent-node).

**Retry a flaky step.** `ai` node with `on_failure_node` pointing at itself, and `on_success_node` to the next step. Give it a `timeout_minutes` so a hung run gets killed rather than looping. If it crashes rather than reporting failure, the run halts for inspection.

**Sweep N targets.** `trigger` -> `workflow_call` (`target_workflow: "Fix One Repo"`, `wait_for_completion: true`, `config: {"await_count": 8}`) -> `ai` (summarize). The parent parks in `awaiting` until all eight finish, then routes on whether the group came back `completed` or `completed_with_failures`.

**Wait for something external.** `ai` (kick off a deploy) -> `sleep` (5 min) -> `shell` (poll the health endpoint), with the shell node's failure edge looping back to the `sleep`. Non-zero exit means not healthy yet, so it sleeps and polls again.

## Gotchas

- A crash does not take the failure edge. Only a reported `task_result` of `failure` does.
- `ai` nodes must write the `task_done` marker atomically, or the watcher may read a partial file. Use `.tmp` then `mv`.
- Questions must go through the `AskUserQuestion` tool. Prose questions do not block.
- `await_count` is capped at 10.
- Only node-level `timeout_minutes` is enforced; the workflow-level timeout is advisory text in `WORKFLOW.md`. There is no budget cap or automatic retry.
- `shell` node scripts live in the `prompt` column.
- The workspace starts empty. Clone the repo yourself in an early node.
- Git operations (branch / commit / PR) are prompt-driven; there are no workflow git flags.
- Deleted nodes are soft-deleted (`deleted_at`) so old sessions keep resolving. A default scope hides them; use `.with_deleted` when you need history.
- Hidden workflows are soft-deleted too (`hidden_at`), and cannot be picked as a `workflow_call` target.
