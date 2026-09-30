# Running Across Multiple Machines

CoiPond is built to run as a control plane plus one or more worker nodes, all sharing one PostgreSQL database. This page covers the fleet model, per-node configuration, the process layout, running the control plane in a coi container, and operating the fleet.

On a single machine none of this applies - the defaults make everything run in one place, exactly as before. This page is only needed when you add a second machine.

## The Model

- **Control plane** - the web UI, the Solid Queue *dispatcher* (which enqueues recurring jobs), and the fleet views (<code>/admin/status</code>). It can run in a container without Incus.
- **Worker node** - a machine with Incus + the `coi` CLI that actually launches and monitors session containers. You can have as many as you like.
- **Shared Postgres** - the one piece every machine talks to. It is the coordination substrate: slot allocation (`pg_advisory_xact_lock`), the node registry, and job queues all live here. There is no node-to-node networking required for orchestration.

An execution is **pinned to the node that first ran it**, because its workspace lives on that node's disk. All of a session's lifecycle jobs (spawn, timeout, wake) route back to the owning node.

## Per-Node Configuration

Set these on each worker node:

<table>
<thead>
<tr><th>Env var</th><th>Purpose</th></tr>
</thead>
<tbody>
<tr><td><code>COIPOND_NODE</code></td><td>Stable identifier for this node (defaults to the OS hostname). Placement, job routing, terminal streaming, and the health registry all key on it. Set it when the OS hostname is ephemeral.</td></tr>
<tr><td><code>COIPOND_QUEUES</code></td><td>Which Solid Queue queues this node's worker consumes (comma/space separated). Unset means <code>"*"</code> (all queues) - correct for a single machine, <strong>wrong for a fleet</strong> (a node would grab other nodes' session jobs).</td></tr>
<tr><td><code>COIPOND_PUBLIC_URL</code></td><td>This node's externally-reachable base URL (e.g. <code>http://node-1:3000</code>) that browsers use to stream a session's terminal directly from this node. The node self-reports it into the registry via its heartbeat; the control-plane UI points the terminal at it automatically (you never enter a node URL). Defaults to <code>http://&lt;primary-IP&gt;:&lt;COIPOND_PUBLIC_PORT or 3000&gt;</code>; set it explicitly behind NAT or with multiple interfaces.</td></tr>
</tbody>
</table>

### The Queue String

The queues CoiPond uses:

- `sessions:<node>` - a session's spawn/timeout/wake/grace-finalize jobs, routed to the owning node (they touch that node's container).
- `sessions:unplaced` - first-spawn jobs for executions not yet pinned to a node; whichever worker picks one up places the execution on itself.
- `incus` - jobs that need Incus but aren't tied to a session (image builds). Any worker node.
- `default` - **control-plane-safe** maintenance: stuck-execution cleanup, dead-node reaping, completion checks. Pure DB, no container/Incus access - runnable anywhere.
- `triggers` - scheduled-trigger checks. `webhooks` - inbound webhook processing. Both DB-only.

On each **worker node** set, for example:

```shell
COIPOND_NODE=box-1
COIPOND_QUEUES="sessions:box-1,sessions:unplaced,incus,default,triggers,webhooks"
```

The **control plane** (no Incus) consumes only the safe queues - e.g. `COIPOND_QUEUES="default,triggers,webhooks"` - never `sessions:*` or `incus`.

Each node consumes **its own** `sessions:<node>` (never another node's), plus the shared `sessions:unplaced` and the non-session queues. A job on a shared queue is still run exactly once - whichever worker claims it first - so having every node subscribe to `default`/`triggers`/`webhooks` does not duplicate work.

## Per-Node Processes

Run these on **each worker node** (see `Procfile.dev` for the dev equivalents):

- `agent` - `bin/node_agent`: the per-node agent, one process that runs both node-owned daemons - session lifecycle monitoring (markers + panes) and health reporting. (You can still run them separately as `bin/session_monitor` + `bin/node_heartbeat` if you prefer.)
- `worker` - `bundle exec rake solid_queue:start` (honors `COIPOND_QUEUES`). Kept separate: Solid Queue's supervisor is fork-based and meant to run standalone.
- `web` - a Falcon instance, because the browser streams a session's terminal directly from the node that owns its container (token-authed, cross-origin). The control-plane UI links to the owning node's terminal endpoints.

On the **control plane** run `Procfile.controlplane` - `web` (the UI + terminal front) plus a Solid Queue process scoped to the safe queues (which also runs the dispatcher for recurring jobs). It needs **no Incus**.

## Running the Control Plane in Incus (via coi)

The control plane needs no Incus of its own, so it doesn't have to run on a bare host or in Docker. Because every CoiPond deployment already has Incus + coi, the cleanest option is to run the control plane as a **persistent coi container** - no extra runtime to install, and no dependency on a public Ruby Docker base image.

The control plane is a plain, long-lived Ruby web app in the container - it never launches session containers (nodes do), so there's **no nested Incus**. It's launched with **coi directly, not via CoiPond**, using a static profile checked in at `deploy/control-plane.toml`:

- **Persistent + port-publishing.** The profile marks the container `persistent = true` (state lives under `/workspace`) and publishes the ports the fleet needs **from the profile itself** - coi wires Incus proxy devices for each `[[ports.map]]` entry, so **no manual `incus config device add`**:

```toml
[[ports.map]]
name = "web"          # UI + terminal front, for browsers
container = 3000
host = 3000
listen = "0.0.0.0"    # LAN-reachable
```

A second `db` entry publishes 5432 for nodes when Postgres runs in this container; drop it if Postgres lives elsewhere and point everything at it via `DB_HOST`/`DB_PORT`.

Then:

```shell
cd /path/to/coi-pond
coi shell --profile deploy/control-plane.toml     # coi auto-mounts the checkout to /workspace
# inside, once: bundle install && yarn install; yarn build && yarn build:css; bin/rails db:prepare
foreman start -f Procfile.controlplane
```

Persistence comes entirely from the persistent profile + `/workspace` storage.

!!! warning "Validate the coi launch on a real host"
    the coi launch, persistence, and port-publishing can't be exercised without Incus.

## Trying It on One Machine (Fleet Mode)

To validate the multi-machine path without a second box, run a single machine configured exactly like a fleet member - explicit `COIPOND_NODE` and a scoped `COIPOND_QUEUES` instead of the single-machine `"*"`. `Procfile.node` wires this up:

```shell
COIPOND_NODE=node-1 foreman start -f Procfile.node --env /dev/null
```

This exercises the real routing/registry/placement/drain/reaping paths (one node consumes only `sessions:node-1` + the shared queues, and reports itself into the registry) and confirms the scoped queue list starves no jobs. What it does *not* exercise is the cross-origin terminal transport (single web, shared disk streams same-origin) or physical node isolation - those need genuinely separate machines.

## Images

Incus images are **local to each machine's Incus daemon** - there is no shared image store by default. Build `coi-default` (and any custom images) on **every node**:

```shell
rake incus:build
```

A node missing its image surfaces as an unhealthy `images_available` check on that node's card in <code>/admin/status</code> rather than launching broken sessions.

## Operating the Fleet

- **<code>/admin/status</code>** shows a **Fleet Nodes** card per node (Live / Stale / Draining + each node's reported checks).
- **Drain** a node (button on its card) to stop the scheduler placing new executions on it while its in-flight work finishes - e.g. before maintenance.
- A node that stops reporting for longer than the dead threshold has its stranded executions failed automatically (they can't be moved - their workspace is on the dead node's disk), freeing capacity.
- **Terminal auth:** the node terminal endpoints (`terminal_sse`/`terminal_input`/`terminal_resize`) always require a valid, session-scoped, expiring capability token minted by the control plane - every node, local or remote, the same. The browser sends it and refreshes it automatically, so long-lived streams and input keep an unexpired token.

## Queue Split (Control-Plane-Safe vs Incus)

Jobs are separated by whether they touch a container/Incus, so the control plane can run a worker without ever picking up an Incus job:

- **Container/Incus jobs are node-only:** image builds go to `incus`; grace finalization (which kills the session's tmux/container) routes to the owning node's `sessions:<node>`, alongside spawn/timeout/wake.
- **`default`/`triggers`/`webhooks` are control-plane-safe** (pure DB) and can run anywhere.

So the control plane runs `web` + a worker on `default,triggers,webhooks` (and the dispatcher for recurring jobs); worker nodes run `sessions:<node>,sessions:unplaced,incus` (and may also share `default`/`triggers`/`webhooks`). A control plane with no Incus can never be handed a job it can't run.
