# How It Works

CoiPond runs as a control plane plus one or more worker nodes over a shared database, launching each agent session in its own isolated Incus container. This page walks through the architecture, the lifecycle of a single session, and how isolation works.

## The Big Picture

CoiPond runs as a **control plane** plus one or more **worker nodes**, all sharing a single PostgreSQL database. On a single machine the two roles collapse into one process group and you do not have to think about the split; it becomes relevant only when you add a second machine.

- **Control plane** - the web UI, the job dispatcher (which enqueues recurring trigger and maintenance jobs), and the fleet views. It needs no Incus of its own.
- **Worker node** - a machine with Incus + the `coi` CLI that actually launches and monitors session containers. You can have as many as you like.
- **Shared PostgreSQL** - the coordination substrate. Slot allocation (via `pg_advisory_xact_lock`), the node health registry, and the job queues all live here, so there is **no node-to-node networking required** for orchestration.

An **execution is pinned to the node that first ran it**, because its workspace lives on that node's disk. All of a session's lifecycle jobs (spawn, timeout, wake, finalize) route back to the owning node through node-scoped job queues.

## A Session's Life

1. **Trigger fires** - a cron tick, an inbound webhook, the completion of another workflow, or a manual click creates a workflow **execution**.
2. **Placement** - the execution claims a free **slot** (lane) on a node. Slots cap how many runs a workflow can have in flight; excess triggers queue.
3. **Container up** - the owning node renders a per-session [coi profile](How-CoiPond-Uses-Coi) and starts an Incus container with `coi shell`, binding the execution's workspace to `/workspace`.
4. **Agent launch** - CoiPond asks the resolved agent adapter (or `coi` itself, for coi-native tools) for the exact launch command, then runs the agent inside a host-side `tmux` pane wrapping `coi container exec`.
5. **Monitoring** - a per-node daemon tails the session's output and watches a filesystem "marker" directory for lifecycle signals: task done, a blocking question, an idle/crash backstop.
6. **Human-in-the-loop (if needed)** - if the agent asks a question, the session flips to *blocked* and surfaces the prompt in the UI; an operator (or a configured rule) answers, and the session resumes.
7. **Completion and routing** - when the agent finishes, the session is finalized and the workflow advances along its success/failure edges to the next node (which may be another agent, a shell step, an approval gate, or a sub-workflow).
8. **Teardown** - non-persistent containers are deleted; persistent-node containers are stopped and kept so the next run resumes the same conversation.

## Isolation

Every session gets its own Incus container. CoiPond never talks to `incus` directly - it shells out to the **`coi`** CLI for every container operation. What a container can see is decided per session by a coi profile (a TOML document): which image to boot, what to mount, the [network policy](Network-Profiles), git identity, timezone, and agent/tool settings. This means the blast radius of any single agent run is one disposable container with exactly the access you granted it.
