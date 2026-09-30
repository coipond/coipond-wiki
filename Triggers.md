# Triggers

A trigger is what starts a workflow. CoiPond supports manual runs, cron schedules, inbound webhooks, and chaining from another workflow's completion.

## Manual

The Run button in the UI. A manual trigger takes **no parameters**: `TriggerManually` sets `trigger_context` to `{triggered_by: "manual"}` and nothing else. If a run needs an input (which repo, which doc), either bake it into the node prompts, use a webhook trigger and read the payload from `TRIGGER.md`, or start with an `interactive` node and type it.

## Cron and Scheduled

Set a schedule on a trigger node. The control-plane dispatcher enqueues due checks, so scheduled workflows fire without any external scheduler. Standard cron expressions work (for example `0 2 * * MON` for 02:00 every Monday). Internally the `trigger_mode` is `scheduled`, and `Triggers::CheckScheduled` picks up rows where `next_check_at <= now`.

## Webhook

Every webhook trigger gets its own `webhook_token` on create, giving it a private, tokenized URL at `POST /api/webhooks/trigger/<trigger-token>`. POST any JSON and the payload is available to the run:

```shell
curl -X POST https://your-host/api/webhooks/trigger/<trigger-token> \
  -H 'Content-Type: application/json' \
  -d '{"pull_request": {"number": 42, "head": {"ref": "feature-x"}}}'
```

There is also a generic endpoint, `POST /api/webhooks/ingest`, for a single shared inbound stream. Point a GitHub (or any) webhook at the trigger URL; the delivered fields become part of the run's context. The payload lands in `TRIGGER.md` in the workspace, so a node's prompt can reference the delivered fields instead of hardcoding values.

## Chaining and Workflow Call

A workflow can trigger another on completion, letting you build larger pipelines out of smaller workflows. `Workflow#trigger_workflow` links one workflow to another, and a [workflow_call](Node-Types) node invokes another workflow directly. See [Workflows](Workflows) for fan-out, fan-in, and how the calling session waits on a batch of child runs.
