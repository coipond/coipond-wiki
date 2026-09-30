# FAQ

Short answers to the questions people ask most often about CoiPond.

## Questions and Answers

- **Do I have to use Claude?** No. CoiPond is agent-agnostic; Claude Code and Aider ship today, behind a small adapter layer. See [Architecture](../Internals/Architecture.md).
- **Does anything leave my machines?** Only the agent's own API calls (e.g. to the model provider). There is no CoiPond SaaS; orchestration state lives in your PostgreSQL.
- **Can it run on a single machine?** Yes - that is the default. The control-plane/worker split only matters once you add a second machine.
- **What starts a workflow?** A manual click, a cron schedule, an inbound webhook, or the completion of another workflow. See [Triggers](../Building-Workflows/Triggers.md).
- **Where do workflows live?** As CNCF Serverless Workflow v1.0 documents, so they are portable and version-controllable.
