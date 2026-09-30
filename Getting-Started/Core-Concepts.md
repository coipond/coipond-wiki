# Core Concepts

These are the building blocks CoiPond composes: workflows and their nodes, the triggers that start them, and the executions, sessions, and slots that run them.

<table>
<thead>
<tr><th>Concept</th><th>What It Is</th></tr>
</thead>
<tbody>
<tr><td><strong>Workflow</strong></td><td>A named pipeline of nodes, stored as a CNCF Serverless Workflow v1.0 document. Owns settings (system prompt, timeouts, concurrency, persistence).</td></tr>
<tr><td><strong>Node</strong></td><td>One step in a workflow. Node types: <strong>ai</strong> (run an agent on a prompt), <strong>shell</strong> (run a script), <strong>interactive</strong> (a human-driven agent session), <strong>approval</strong> (a gate that waits for a click), <strong>sleep</strong> (a timed pause), <strong>trigger</strong> (an entry point: cron / webhook / manual), and <strong>workflow_call</strong> (invoke another workflow).</td></tr>
<tr><td><strong>Trigger</strong></td><td>What starts a workflow: a cron schedule, an inbound webhook (with a per-trigger token URL), the completion of another workflow, or a manual run.</td></tr>
<tr><td><strong>Execution</strong></td><td>One run of a workflow, pinned to the node that first placed it. Holds the workspace and walks the node graph.</td></tr>
<tr><td><strong>Session</strong></td><td>One agent (or shell) run inside a container - the unit that streams a terminal and produces a result. A node produces one session per execution.</td></tr>
<tr><td><strong>Slot (lane)</strong></td><td>A concurrency unit per workflow (1..10). Slots cap parallel executions; each lane is an isolated container.</td></tr>
<tr><td><strong>coi profile</strong></td><td>The per-session TOML that tells <code>coi</code> how to build the container (image, mounts, network, git, tool config). Rendered fresh per session from the node's base profile plus session overlays.</td></tr>
<tr><td><strong>Network policy</strong></td><td>The <code>[network]</code> section of a coi profile (open / restricted / allowlist), applied by coi when the container launches. Not a separate object - it lives in the coi profile.</td></tr>
<tr><td><strong>Node (machine) / registry</strong></td><td>A worker machine in the fleet. Each self-reports health, image completeness, and its reachable URL into a shared registry via a heartbeat.</td></tr>
</tbody>
</table>

See also [Node Types](../Building-Workflows/Node-Types.md) for what each node does in depth, [Triggers](../Building-Workflows/Triggers.md) for how workflows start, and [Workflows](../Building-Workflows/Workflows.md) for the routing mechanics.
