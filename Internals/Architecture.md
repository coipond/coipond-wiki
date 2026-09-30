# Architecture

CoiPond is organized as a Rails application built from small, single-purpose objects. This page covers the tech stack it runs on and how the domain logic is laid out.

## Tech Stack

CoiPond runs on Ruby 4.0, Rails 8.1, and PostgreSQL 18, with Solid Queue/Cache/Cable for background work and the Falcon web server. The front end pairs Hotwire with React 19 and xterm.js, styled with Tailwind CSS 4 and DaisyUI 5. Containers run via the [coi](https://github.com/mensfeld/code-on-incus) CLI (Incus plus tmux).

The application is organized as a set of small, single-purpose objects under `app/concepts/` (a lightweight domain-driven layout), rather than fat models and service objects.

## Project Layout

CoiPond favors small, single-purpose objects over fat models and service classes. The bulk of the domain logic lives under `app/concepts/`, where the first path segment is a namespace (Zeitwerk-loaded):

- `app/concepts/workflows/` - workflows, nodes, triggers, executions, sessions, and the session lifecycle (monitoring, markers, logging, the CNCF definition serializer/parser).
- `app/concepts/containers/` - the coi integration: `Commands` (the CLI wrapper), profile rendering, container setup/teardown, and the session launch pipeline (prepare, launch tmux, resource sampling).
- `app/concepts/agents/` - the agent adapter layer (`ClaudeCode`, `Aider`) behind a `Registry`.
- `app/concepts/coi_pond/` - the node fabric: the node identity/registry, the per-node agent and its daemons (monitoring, heartbeat, image reconciliation), and the node-to-node capability tokens.

Operations follow a small step-based pattern (`ApplicationOperation`) or a stateless `Callable` interface, and value objects are plain keyword-initialized `Struct`s.
