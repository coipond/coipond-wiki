# Troubleshooting

Common failure modes and how to resolve them. Each item leads with the symptom you see, then the fix.

## Problems and Solutions

- **A run never starts / sits queued.** Check <code>/admin/nodes</code>: is a node Live? Is its `images_available` check healthy? A missing `coi-default` image or a drained/stale node will hold placements. Build the image (`rake incus:build`) and confirm the node is reporting.
- **Sessions fail immediately.** Confirm the `coi` binary is on `PATH` and new enough - <code>/admin/status</code> flags an out-of-date binary. `COI_BINARY_PATH` overrides the location.
- **The agent cannot reach an API it needs.** Its coi profile's `[network]` allowlist is probably missing that domain (or `8.8.8.8`/`1.1.1.1` for DNS). Add the domain to the profile. See [Network Profiles](Network-Profiles).
- **A secret is not visible inside the container.** Its **name** must be in the profile's `forward_env`, and its **value** must exist in CoiPond's env-var store at a matching scope. Host env vars are never blanket-forwarded. See [Configuration](Configuration).
- **Terminal is blank on a fleet.** The owning node must have a reachable `COIPOND_PUBLIC_URL` (it self-reports it); behind NAT or with multiple interfaces, set it explicitly.
- **Jobs are not being picked up.** On a fleet, check `COIPOND_QUEUES`: a node must consume its own `sessions:<node>` plus the shared queues, and the control plane must consume `default,triggers,webhooks`. Unset (`"*"`) is correct only on a single machine. See [Running Across Multiple Machines](Multiple-Machines).
