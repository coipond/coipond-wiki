# How CoiPond Uses Coi

CoiPond runs every session inside a code-on-incus (coi) container and drives coi entirely through its CLI. This page covers the integration contract: the command surface, the per-session profile, the container lifecycle, and the coi features CoiPond leans on.

## Overview

CoiPond uses code-on-incus (coi) v0.12+ as its container runtime layer. coi provides isolated Incus containers with built-in network isolation, SSH agent forwarding, resource limits, security hardening, and AI tool configuration.

CoiPond drives coi entirely through the CLI (wrapped in `Containers::Commands`, `app/concepts/containers/commands.rb`) - it never talks to `incus` directly. A session's container is described by a per-session coi profile (TOML) and launched with `coi shell --profile`; coi applies the image, mounts, network policy, security hardening, git identity, timezone, and tool config from that profile in one step.

## Architecture

The primary path is profile-driven:

1. CoiPond renders a per-session coi profile from the node's base profile plus session overlays (`Containers::CoiProfileRenderer` builds the TOML; `Containers::CoiProfileGenerator` writes it to `~/.coi/profiles/<name>/config.toml`).
2. It starts the container with `coi shell --profile <name>`. coi reads the profile and applies the image, workspace/storage mounts, network policy, security, git identity, timezone, and tool config in one go. CoiPond parses the container name coi allocated from the `--debug` output.
3. It launches the agent with `coi container exec` inside a host-side `tmux` pane. For coi-native tools (Claude), CoiPond asks `coi tool spec` for the exact launch argv plus tool env, so coi stays the source of truth for the model and permission settings.

Supporting commands: `coi file push` (session data / config files), `coi image ...` (custom image build/publish), and `coi top --json` (resource sampling for the per-node Top views). `Containers::Commands` also retains lower-level `container launch/stop/delete/exec/configure/mount` wrappers used by image builds and maintenance paths.

## Container Lifecycle

### Session Flow

The `Containers::Setup` pipeline runs these steps:

```text
1. determine_image             (CoiPond: node/workflow -> image alias)
2. determine_execution_context (CoiPond: home dir, container user)
3. generate_profile            (CoiPond: render per-session coi profile TOML -> ~/.coi/profiles/<name>)
4. launch_via_coi_shell        (coi: `coi shell --profile <name>` - applies image, mounts, network,
                                security, git, timezone, tool config; CoiPond parses the container name)
5. verify_workspace_mount      (CoiPond: confirm /workspace is bound)
6. restore_session_data        (coi: `coi file push` - restore prior state for persistent nodes)
7. install_agent_config        (CoiPond: agent-specific config files into the container)
```

The session is then prepared (env vars, prompt, context files), the agent is launched in tmux via `coi container exec` (argv from `coi tool spec` for coi-native tools), monitored (log tailing plus marker-based blocking/completion detection), and finally torn down: non-persistent containers are stopped and deleted; persistent-node containers are stopped and kept so the next run resumes.

## The Coi Profile

Everything about a container is expressed in its coi profile TOML - image, resource limits, security, mounts, network policy, git identity, timezone, and tool config. Operators author profiles as first-class objects (managed under Coi Profiles in the UI); a node picks one, and nodes that do not fall back to the seeded default profile. Session-time overlays (the workspace/storage mounts, git identity, image alias, `persistent` flag, control-plane env) are appended per session by the renderer - operators do not encode those.

A minimal profile:

```toml
[limits.cpu]
count = "4"

[limits.memory]
limit = "8GiB"

[security]
additional_protected_paths = [".git/hooks"]

[tool]
name = "claude"
permission_mode = "bypass"

[incus]
code_user = "code"
```

Add a `[network]` block to lock down egress - see [Network Profiles](../Configuration/Network-Profiles.md).

## Commands Used

Launch a session container from a profile (primary path):

```shell
coi shell --profile <name> --debug        # applies image + mounts + network + security + tool config
```

Execute the agent (PTY, user context, env vars) inside the container:

```shell
coi container exec -t <name> --user 1000 --cwd /workspace --env KEY=VAL -- <command>
```

Execute with output capture (probes, readbacks):

```shell
coi container exec <name> --capture --format=raw --cwd / -- <command>
```

The launch argv/env for a coi-native tool:

```shell
coi tool spec --json ...
```

Files:

```shell
coi file push <local> <container>:<remote>
```

Images:

```shell
coi image list --format=json [--prefix P]
coi image exists <alias>
coi image publish <container> <alias> [--description TEXT]
```

Resource sampling (per-node Top):

```shell
coi top --json [--procs]
```

## Features Leveraged From Coi

All of these are configured through the profile TOML (applied at `coi shell` launch), not separate post-boot flags:

- **Network isolation** - nftables enforcement from the profile's `[network]` section. Modes: open (no block), `restricted` (blocks RFC1918 plus metadata endpoint), `allowlist` (only `allowed_domains`). See [Network Profiles](../Configuration/Network-Profiles.md).
- **SSH agent forwarding** - coi forwards the host `SSH_AUTH_SOCK` into the container as a proxy device; CoiPond falls back to a DB-stored key only when no host agent is available.
- **Resource limits** - CPU / memory / disk I/O / max-processes / tmpfs, from the profile's `[limits.*]`.
- **Security hardening** - coi read-only-mounts host-auto-executing sinks (`.claude/settings.json`, `.git/config`, `.husky`) so a contained agent cannot plant a payload the host later runs; the profile's `[security]` can add more protected paths.
- **Git identity guard** - coi sets `user.useConfigOnly=true` so the agent cannot commit as the container's default user; CoiPond overlays the host git identity per session.
- **Tool config** - `[tool]` (e.g. `name = "claude"`, `permission_mode`, model) is applied by coi as container-level env inherited by every exec, so the tool's model/permission settings come from the profile (requires coi with [#745](https://github.com/coipond/coi/pull/745); the System Status page flags an out-of-date binary).
- **Timezone** - the host timezone is overlaid into the profile per session.

## CoiPond-Specific Logic

These stay in CoiPond because they are tied to the workflow engine and are not delegated to coi:

1. **Session management** - session ids, resume/`--continue` logic, state persistence on persistent nodes.
2. **Prompt building** - TRIGGER.md / WORKFLOW.md context files and prompt construction.
3. **tmux orchestration** - host-side tmux panes wrapping `coi container exec`, and the terminal stream.
4. **Monitoring** - pane/log tailing and marker-based blocking / completion / idle detection.
5. **Environment variables** - a single flat env-var namespace (name -> encrypted value); a per-profile `forward_env` allow-list controls which names cross into the container.
6. **Agent config** - session-end hooks, workspace trust, permission bypass.
7. **Custom image building** - user-provided build scripts via the UI.

## Configuration Files

<table>
<thead>
<tr><th>File</th><th>Purpose</th></tr>
</thead>
<tbody>
<tr><td><code>app/concepts/containers/commands.rb</code></td><td>Central coi CLI wrapper</td></tr>
<tr><td><code>app/concepts/containers/coi_profile_renderer.rb</code></td><td>Renders the per-session profile TOML (pure)</td></tr>
<tr><td><code>app/concepts/containers/coi_profile_generator.rb</code></td><td>Writes the rendered profile to disk plus mount-dir side effects</td></tr>
<tr><td><code>app/concepts/containers/setup.rb</code></td><td>Container launch pipeline (<code>coi shell --profile</code>)</td></tr>
<tr><td><code>app/concepts/containers/sessions/prepare.rb</code></td><td>Env vars, prompt, launch command</td></tr>
<tr><td><code>app/concepts/containers/sessions/launch_tmux.rb</code></td><td>tmux session wrapping <code>coi container exec</code></td></tr>
<tr><td><code>app/models/coi_profile.rb</code></td><td>The coi profile model (the raw TOML operators author)</td></tr>
</tbody>
</table>

### Coi Binary Path

```text
ENV["COI_BINARY_PATH"]  (default: "coi" on PATH)
```

## Deployment Requirements

- **coi v0.12+** (see the version note in the README).
- **Incus** 6.1 or newer, installed and configured (see the coi [Linux Setup Guide](https://github.com/coipond/coi/wiki/Linux-Setup-Guide)).
- **nftables** (`nft` plus passwordless sudo for it), for network isolation (see the coi [nftables Setup](https://github.com/coipond/coi/wiki/nftables-Setup)).
- **SSH agent** on the host (optional, for SSH agent forwarding).

## Version History

<table>
<thead>
<tr><th>Date</th><th>Change</th></tr>
</thead>
<tbody>
<tr><td>2026-09</td><td>Rewritten for the profile-driven flow: <code>coi shell --profile</code>, per-session rendered profiles, <code>coi tool spec</code>; network/SSH/limits/tool config all expressed in the profile TOML. The standalone NetworkProfile model and the sandbox/privileged split were removed.</td></tr>
<tr><td>2026-05</td><td>coi v0.8+: profiles, configure command, network enforcement, SSH forwarding, resource limits.</td></tr>
<tr><td>2026-01</td><td>Original document: low-level container commands.</td></tr>
</tbody>
</table>
