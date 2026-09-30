# Network Profiles

Network access for a session's container is controlled by the `[network]` section of the coi profile assigned to the node that runs it. This page covers the three modes, how to set a policy, worked examples, and best practices.

There is no separate "network profile" object to manage; network policy is one part of the coi profile, alongside resource limits, security, mounts, and tool config.

coi enforces the policy at the container's network layer (firewalld/nftables), so it holds regardless of what the agent tries to do.

## Modes

The `[network]` block supports three modes:

- **open** - unrestricted access. This is coi's default, so a profile with **no `[network]` block** is open.
- **restricted** - blocks RFC1918 private ranges (`10/8`, `172.16/12`, `192.168/16`) and the cloud metadata endpoint (`169.254.169.254`), but allows the public internet. Use it when a run needs the internet but must not reach your internal network.
- **allowlist** - only the named domains are reachable; everything else is blocked. Best for credential-sensitive runs (it minimizes where a leaked token could be sent).

## Setting It

Network policy lives in a coi profile, which you manage under **Coi Profiles** in the UI (or seed on disk). Two ways to apply a policy to a workflow:

1. **Give the node a profile that has the `[network]` block you want.** Each workflow node can pick a coi profile; nodes that do not pick one fall back to the seeded **default** profile (which is open - no `[network]` block).
2. **Edit the profile TOML directly** to add or change the `[network]` section, then assign that profile to the node(s).

Because the whole profile is just TOML, you can keep a small library of profiles (for example an `open` profile and a locked-down `github-anthropic` allowlist profile) and pick the right one per node.

## Example: A GitHub + Anthropic Allowlist Profile

```toml
[network]
mode = "allowlist"
allowed_domains = [
  "github.com",
  "api.github.com",
  "githubusercontent.com",
  "api.anthropic.com",
  "8.8.8.8",   # DNS
  "1.1.1.1",   # DNS
]

[tool]
name = "claude"
permission_mode = "bypass"
```

Assign this profile to a node and the agent can reach GitHub and the model API and nothing else; any other request is blocked.

## Example: Restricted (Internet, But No Internal Network)

```toml
[network]
mode = "restricted"
```

## Best Practices

- **Default to allowlist in production.** It is the single biggest lever on blast radius and credential safety; reserve `open` for experimentation.
- **Include DNS in allowlists.** Add `8.8.8.8` and `1.1.1.1` (or your resolver) so name resolution works.
- **One profile per external service.** Keep focused profiles (a GitHub profile, an internal-API profile) and compose your fleet by assigning the right one per node.
- **Pair with `forward_env`.** The network policy limits where a run can talk; `forward_env` (also in the profile) limits which secrets it even has. See [Configuration](Configuration) for how the env-var store and `forward_env` work. Use both for credential-sensitive automation.

## Notes

- Network policy, like everything else in a coi profile, is applied when the session container is launched via `coi shell --profile`; there is no separate configuration step or in-workspace config file.
- coi caches DNS resolutions for allowlist mode.
- The seeded system profile deliberately ships **without** a `[network]` block (open) so first runs work out of the box; lock it down by assigning a stricter profile.
