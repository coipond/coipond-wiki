# Tips and Best Practices

Practical guidance for authoring workflows that finish cleanly, stay isolated, and keep secrets least-privilege. Each takeaway leads with the practice.

- **Prompt for a clean finish.** Autonomous AI nodes work best with a clear definition of done. End prompts with an explicit stop condition so the completion detector fires promptly.
- **Lock down the network.** Use a coi profile with an [allowlist](Network-Profiles) `[network]` in production; reserve **open** for experimentation. It is the single biggest lever on blast radius and credential safety.
- **Least-privilege secrets.** Env vars are a single flat namespace; a secret reaches a session only if that session's coi profile lists its **name** in [forward_env](Configuration). Give credential-carrying profiles only to the nodes that need them, and keep untrusted-code nodes on a profile that forwards nothing.
- **Use persistent nodes for warm workspaces.** A node whose profile sets `[container] persistent = true` reuses its container across runs, so `~/.claude` and the checkout survive and the next run resumes the prior conversation (`claude --continue`). Great for iterative, same-repo automation; use ephemeral (the default) when each run should start clean. See [Workflows](Workflows) for how workspace resolution keeps a persistent node on the same directory.
- **Tune concurrency with slots.** `max_concurrent_executions` (1..10) caps a workflow's parallel lanes. Raise it for independent, bursty triggers; keep it low when runs share a scarce resource.
- **Gate the irreversible.** Put an **approval** node before merges and deploys, and route **failure** edges somewhere that notifies you or cleans up rather than leaving a dangling run.
- **Keep prompts idempotent.** Scheduled and webhook runs can fire repeatedly; write instructions that are safe to run again (check before creating a branch or PR, and so on).
- **Build images on every node.** Incus images are local to each machine. A node missing `coi-default` (or a custom image) shows an unhealthy check instead of launching broken sessions.
- **Watch `/admin/status` and `/admin/nodes`.** They surface an out-of-date `coi` binary, missing images, and stale or draining nodes before those turn into failed runs.
