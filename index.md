# CoiPond Wiki

CoiPond is self-hosted infrastructure for orchestrating autonomous AI coding sessions: it triggers coding-agent runs from events, isolates each one in its own [Incus](https://linuxcontainers.org/incus/) container, and chains them into multi-step workflows. Start with [Why CoiPond](Getting-Started/Why-CoiPond.md) for the value proposition, or jump straight to [Getting Started](Getting-Started/index.md).

## Getting Started

- [Why CoiPond](Getting-Started/Why-CoiPond.md) - the value proposition, use cases, and features
- [Getting Started](Getting-Started/index.md) - prerequisites, quick start, and your first workflow
- [How It Works](Getting-Started/How-It-Works.md) - the control-plane / worker model and a session's life
- [Core Concepts](Getting-Started/Core-Concepts.md) - the vocabulary: workflows, nodes, triggers, executions, sessions, slots, profiles

## Building Workflows

- [Workflows](Building-Workflows/Workflows.md) - the workflow behavior reference: routing, results, loops, fan-out, guardrails
- [Node Types](Building-Workflows/Node-Types.md) - the seven node types and their common fields
- [Triggers](Building-Workflows/Triggers.md) - manual, cron, webhook, and chaining
- [Interactive and Blocked Sessions](Building-Workflows/Interactive-and-Blocked-Sessions.md) - live terminals, blocked questions, and approvals
- [Tips and Best Practices](Building-Workflows/Tips-and-Best-Practices.md) - how to get reliable unattended runs

## Configuration

- [Configuration](Configuration/index.md) - environment variables, secrets, and job queues
- [Network Profiles](Configuration/Network-Profiles.md) - open, restricted, and allowlist egress policy

## Operations

- [Admin and Observability](Operations/Admin-and-Observability.md) - the status, fleet, and resource surfaces
- [Deployment](Operations/Deployment.md) - running the control plane and worker nodes
- [Running Across Multiple Machines](Operations/Multiple-Machines.md) - the fleet model, queues, and operations
- [Troubleshooting](Operations/Troubleshooting.md) - common failure modes and fixes
- [FAQ](Operations/FAQ.md) - quick answers

## Internals

- [Architecture](Internals/Architecture.md) - tech stack and project layout
- [How CoiPond Uses Coi](Internals/How-CoiPond-Uses-Coi.md) - the coi integration contract

## Development

- [Development](Development/index.md) - running the suite and linters
- [Security](Development/Security.md) - the isolation and secret-handling model
