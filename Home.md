# CoiPond Wiki

CoiPond is self-hosted infrastructure for orchestrating autonomous AI coding sessions: it triggers coding-agent runs from events, isolates each one in its own [Incus](https://linuxcontainers.org/incus/) container, and chains them into multi-step workflows. Start with [Why CoiPond](Why-CoiPond) for the value proposition, or jump straight to [Getting Started](Getting-Started).

## Getting Started

- [Why CoiPond](Why-CoiPond) - the value proposition, use cases, and features
- [Getting Started](Getting-Started) - prerequisites, quick start, and your first workflow
- [How It Works](How-It-Works) - the control-plane / worker model and a session's life
- [Core Concepts](Core-Concepts) - the vocabulary: workflows, nodes, triggers, executions, sessions, slots, profiles

## Building Workflows

- [Workflows](Workflows) - the workflow behavior reference: routing, results, loops, fan-out, guardrails
- [Node Types](Node-Types) - the seven node types and their common fields
- [Triggers](Triggers) - manual, cron, webhook, and chaining
- [Interactive and Blocked Sessions](Interactive-and-Blocked-Sessions) - live terminals, blocked questions, and approvals
- [Tips and Best Practices](Tips-and-Best-Practices) - how to get reliable unattended runs

## Configuration

- [Configuration](Configuration) - environment variables, secrets, and job queues
- [Network Profiles](Network-Profiles) - open, restricted, and allowlist egress policy

## Operations

- [Admin and Observability](Admin-and-Observability) - the status, fleet, and resource surfaces
- [Deployment](Deployment) - running the control plane and worker nodes
- [Running Across Multiple Machines](Multiple-Machines) - the fleet model, queues, and operations
- [Troubleshooting](Troubleshooting) - common failure modes and fixes
- [FAQ](FAQ) - quick answers

## Internals

- [Architecture](Architecture) - tech stack and project layout
- [How CoiPond Uses Coi](How-CoiPond-Uses-Coi) - the coi integration contract

## Development

- [Development](Development) - running the suite and linters
- [Security](Security) - the isolation and secret-handling model
