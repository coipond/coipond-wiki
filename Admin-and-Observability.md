# Admin and Observability

CoiPond exposes a set of admin surfaces for system health, fleet state, per-node resource usage, and the job queue dashboard. These are where you watch the platform and catch problems before they turn into failed runs.

## Admin Surfaces

- <code>/admin/status</code> - system status: coi binary version, image readiness, infrastructure checks.
- <code>/admin/nodes</code> - the fleet: a card per node (Live / Stale / Draining) with its reported health, plus **Drain**/**Resume** controls.
- <code>/admin/nodes/&lt;node&gt;/top</code> - per-node "coi top": live container and process resource usage for that node.
- <code>/jobs</code> - the Mission Control dashboard for Solid Queue (basic-auth protected).

For fleet operations, draining nodes, and how the registry reports health across machines, see [Running Across Multiple Machines](Multiple-Machines).
