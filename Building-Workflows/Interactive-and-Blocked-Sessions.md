# Interactive and Blocked Sessions

CoiPond keeps unattended runs safe by pulling a human in only when one is needed. This page covers the three human-in-the-loop mechanisms: live terminals you drive yourself, blocked sessions where an agent asks a question mid-run, and explicit approval gates.

## Live Terminals

Any session streams a live xterm.js terminal in the browser over Server-Sent Events. For **interactive** nodes you drive the agent yourself, exiting the pane with `//success` or `//failure`; for autonomous nodes you can still watch, and type to intervene.

Terminal streaming and input are authenticated with a short-lived, session-scoped capability token minted by the control plane, so the browser can stream directly from whichever node owns the container. This works the same on one machine or a fleet.

## Blocked Sessions

When an `ai` node hits something it genuinely cannot decide, the session flips to **blocked** and the prompt appears in the UI for a human (or a configured rule) to answer; answering resumes the run.

!!! warning "Only the `AskUserQuestion` tool pauses a run"
    a blocked session happens only if the agent calls the `AskUserQuestion` tool. Printing a question as prose does not pause anything - the session finalizes on the next Stop. The `session_automated` prompt is emphatic about this.

`Session::NEEDS_REVIEW_STATUSES` is `blocked` plus `pending_approval`, and backs the "needs attention" surfaces in the dashboard.

## Approval Nodes

An **approval** node is an explicit gate that pauses the workflow until someone clicks through. The session goes `pending_approval` and waits for an explicit approve or reject; the node shows its `approval_message` while it waits. Put one before anything irreversible, such as a merge or a deploy.

Together, blocked sessions and approval nodes keep unattended runs safe: the agent proceeds on its own until it hits genuine ambiguity or a guarded step, then waits for you. For where these node types sit in the graph, see [Node Types](Node-Types.md).
