# Configuration

CoiPond is configured through environment variables, a scoped env-var store that controls which secrets reach session containers, and a queue split that keeps container-free machines from picking up container work. This page covers all three.

## Environment Variables

Copy `.env.example` and edit. The essentials:

<table>
<thead>
<tr>
<th>Variable</th>
<th>Required</th>
<th>Purpose</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>DATABASE_URL</code></td>
<td>yes</td>
<td>PostgreSQL connection string (shared by every machine in a fleet).</td>
</tr>
<tr>
<td><code>RAILS_MASTER_KEY</code></td>
<td>yes</td>
<td>Rails credentials key (from <code>config/master.key</code>), or set explicitly.</td>
</tr>
<tr>
<td><code>ANTHROPIC_API_KEY</code></td>
<td>for Claude</td>
<td>API key used by Claude-based agents and sub-agent delegation.</td>
</tr>
<tr>
<td><code>GH_TOKEN</code></td>
<td>optional</td>
<td>GitHub token for git/PR sessions. Falls back to <code>gh auth token</code>.</td>
</tr>
<tr>
<td><code>SSH_PRIVATE_KEY</code></td>
<td>optional</td>
<td>SSH key content for git/SSH-based sessions. Falls back to the host SSH agent / <code>~/.ssh/id_coipond</code>.</td>
</tr>
<tr>
<td><code>COIPOND_USER</code> / <code>COIPOND_PASSWORD</code></td>
<td>optional</td>
<td>HTTP basic-auth for the <code>/jobs</code> (Mission Control) dashboard.</td>
</tr>
<tr>
<td><code>COI_BINARY_PATH</code></td>
<td>optional</td>
<td>Path to the <code>coi</code> binary (defaults to <code>coi</code> on <code>PATH</code>).</td>
</tr>
</tbody>
</table>

> **Multi-machine variables live elsewhere:** `COIPOND_NODE`, `COIPOND_QUEUES`, `COIPOND_PUBLIC_URL`, and `COIPOND_PUBLIC_PORT` are covered in [Running Across Multiple Machines](Multiple-Machines). On a single machine you do not need any of them.

## Secrets That Reach The Container

CoiPond does **not** blanket-forward the host environment into session containers. A container receives only:

- `PATH` (from the container's own login shell) and `HOME`,
- the internal `COIPOND_*` control-plane variables, and
- exactly the variables named in the profile's `forward_env` allow-list, whose **values** come from CoiPond's own env-var store (a single flat namespace of name -> encrypted value).

So a secret is only ever visible to a session if a profile explicitly opts its name in. Nothing else from the host process crosses the boundary. Store the value once in CoiPond's env-var UI, and list its **name** in the `forward_env` of the profiles that should have it. Access is decided by **which profile a node picks**, not by the workflow. See [Network Profiles](Network-Profiles) for how `forward_env` pairs with the network policy in the same coi profile.

## Job Queues

CoiPond separates jobs by whether they touch a container/Incus, so a control plane with no Incus can run a worker without ever being handed a job it cannot execute. `COIPOND_QUEUES` controls which queues a worker consumes; unset means "all queues" (correct for one machine). See [Running Across Multiple Machines](Multiple-Machines) for the full queue model.
