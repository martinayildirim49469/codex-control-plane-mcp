[![MseeP.ai Security Assessment Badge](https://mseep.net/pr/aresyn-codex-control-plane-mcp-badge.png)](https://mseep.ai/app/aresyn-codex-control-plane-mcp)

# Codex Control Plane MCP

English | [Русский](README.ru.md)

<!-- mcp-name: io.github.aresyn/codex-control-plane-mcp -->

[![CI](https://github.com/aresyn/codex-control-plane-mcp/actions/workflows/ci.yml/badge.svg)](https://github.com/aresyn/codex-control-plane-mcp/actions/workflows/ci.yml)
[![PyPI](https://img.shields.io/pypi/v/codex-control-plane-mcp.svg)](https://pypi.org/project/codex-control-plane-mcp/)
[![Python](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![MCP](https://img.shields.io/badge/MCP-stdio-green.svg)](docs/API_CONTRACT.md)

<p align="center">
  <img src="https://raw.githubusercontent.com/aresyn/codex-control-plane-mcp/main/assets/repo_image.png" alt="Codex Control Plane MCP" width="100%">
</p>

Reliable Codex Desktop automation for long tasks.

`codex-control-plane-mcp` turns Codex Desktop and `codex-app-server` into a
durable worker that an MCP client can drive safely. Send a task, get an
`operationId` or `workflowId` right away, poll until the work finishes, approve
Plan Mode when needed, then read the final report.

The server handles the awkward parts that thin wrappers usually leave to the
caller: app-server startup, thread and turn creation, retry safety, duplicate
prompt protection, Plan Mode, approvals, local history, diagnostics, and repair.

OpenClaw and Hermes are first-class clients, but the server is useful for any
local orchestrator that needs Codex Desktop to do long-running work without
holding one MCP call open for hours.

## The short version

```text
MCP client / orchestrator
  -> submit a task or start a Plan Mode workflow
  <- receive operationId or workflowId immediately
  -> poll status
  -> answer approvals or approve the plan
  <- read final report, diagnostics, threadId, and turnId
```

That gives you a simple contract:

- no multi-hour MCP calls;
- no duplicate Codex turns after a client retry;
- no blind fire-and-forget task submission;
- a local SQLite record of operations, workflows, turns, hooks, and diagnostics.

## Why not just call Codex directly?

| Capability | Thin Codex wrapper | Codex Control Plane MCP |
|---|---:|---:|
| Multi-hour tasks | blocking / fragile | durable async operation |
| Client timeout recovery | manual | retry-safe `client_request_id` |
| Duplicate turn protection | no | active prompt detection |
| Plan Mode workflow | human / manual | pollable workflow state |
| Approvals and questions | blocking / opaque | pending interactions API |
| Restart recovery | ad hoc | persisted operation state |
| Diagnostics | logs only | health, diagnostics, repair tools |

For a more detailed decision guide, see
[docs/THIN_WRAPPERS.md](docs/THIN_WRAPPERS.md).

## Current support

- Full live target: Windows with Codex Desktop and `codex-app-server`.
- Linux and macOS: protocol-only checks for now.
- Local-first: not intended to be exposed as a public network service.

## Security model

This is a local-first control plane for trusted Codex Desktop environments.

Do not expose it as a network service without authentication.

Recommended first-run posture:

- use `read-only` for untrusted repositories;
- use `on-request` approval when testing new workflows;
- Plan Mode never runs with a `read-only` sandbox. If a caller requests
  `read-only`, MCP raises that turn to `workspace-write` and reports the
  adjustment in status output;
- keep `state/`, `logs/`, `.env`, and `.codex/` private.

## What it does

- Durable async queue for Codex write operations.
- Retry-safe `client_request_id` handling.
- Active duplicate prompt detection.
- SQLite leases and heartbeats for competing MCP processes.
- Recovery after MCP restart during `thread/start` or `turn/start`.
- Durable `turn/steer` for adding context to an active turn without creating a second turn.
- Durable `thread/fork` for branching an existing thread, with or without an initial message.
- Plan Mode workflows: start plan, poll, approve, execute, read final report.
- Plan Mode runtime floor: `workspace-write`, with `runtimePolicyAdjusted` in
  status when MCP raises a `read-only` request.
- Code review workflows through app-server `review/start`, with polling and final report capture.
- Structured final reports with `output_schema`.
- Thread lifecycle tools for archive, unarchive, and pollable compaction.
- Workflow goal sync with Codex Desktop thread goals.
- Image and local image inputs for turns that start through `turn/start`.
- Pending approvals and questions exposed as pollable MCP state.
- Turn interrupts by `threadId`/`turnId`, `operationId`, or `workflowId`.
- Runtime inventory for models, permission profiles, sandbox readiness, hooks, skills, provider features, account status, usage bands, rate-limit state, and supported app-server methods.
- Health checks, diagnostics, issue analysis, and dry-run repairs.
- MCP-owned hook history in SQLite for search, summaries, and fallback reads.
- Redacted app-server progress journal for deltas, warnings, model reroutes, and token usage.
- Structured MCP errors that automation code can branch on.

Write and control actions go through `codex-app-server`. The server does not
mutate Codex internal SQLite databases or transcript files.

## Install

Recommended:

```powershell
pipx install codex-control-plane-mcp
```

Or run directly:

```powershell
uvx codex-control-plane-mcp
```

From GitHub:

```powershell
python -m pip install "codex-control-plane-mcp @ git+https://github.com/aresyn/codex-control-plane-mcp.git"
```

For local development:

```powershell
git clone https://github.com/aresyn/codex-control-plane-mcp.git
cd codex-control-plane-mcp
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -e ".[dev]"
python -m pytest -q
```

## MCP client config

After installation, generate a config:

```powershell
codex-control-plane-mcp-admin init --state-db .\state\codex-mcp-state.sqlite3 --projects-root C:\Users\you\Projects
```

Minimal stdio entry:

```json
{
  "mcpServers": {
    "codex-control-plane": {
      "command": "codex-control-plane-mcp",
      "args": []
    }
  }
}
```

More copy-paste examples for Claude Desktop, Cursor, VS Code-style MCP clients,
local checkouts, installed packages, and central worker mode are available in
[`examples/mcp-client-configs.md`](examples/mcp-client-configs.md).

Run the MCP stdio server:

```powershell
codex-control-plane-mcp
```

Or run it as a module:

```powershell
py -m codex_control_plane_mcp.server
```

The old `openclaw-codex-mcp` and `openclaw-codex-mcp-hooks` commands remain as
compatibility aliases for one release line.

## Central worker mode

The default `inline` mode is still the simplest setup: one MCP process can submit
and execute operations. For OpenClaw, Hermes, or any setup with several MCP
clients, use a central worker instead.

Recommended local shape:

- every MCP client uses the same `CODEX_HOME` and `CODEX_MCP_STATE_DB`;
- OpenClaw gateway entries run with `CODEX_MCP_EXECUTION_MODE=client`;
- one long-running `codex-control-plane-mcp-worker` process owns
  `codex-app-server`, leases, queue slots, and resource locks;
- clients call `codex_submit_task`, then poll status. They do not execute queued
  operations themselves.

Worker command:

```powershell
$env:CODEX_MCP_EXECUTION_MODE = "worker"
codex-control-plane-mcp-worker
```

Safe observation mode, useful before switching a live gateway:

```powershell
codex-control-plane-mcp-worker --observe
```

Concurrency defaults:

```powershell
CODEX_MCP_MAX_ACTIVE_TURNS_GLOBAL=4
CODEX_MCP_MAX_ACTIVE_TURNS_PER_PROJECT=3
CODEX_MCP_MAX_ACTIVE_TURNS_PER_AGENT=3
CODEX_MCP_MAX_ACTIVE_TURNS_PER_THREAD=1
CODEX_MCP_MAX_ACTIVE_WRITE_TURNS_PER_PROJECT=1
CODEX_MCP_MAX_APP_SERVER_PENDING_REQUESTS=8
```

For write turns in the same project, pass `resource_keys` to
`codex_submit_task`. Without them, `workspace-write` and `danger-full-access`
turns take a broad project write lock. With disjoint keys, the worker may run
several write turns in parallel.

New status tools:

- `codex_get_worker_status`
- `codex_get_queue_status`
- `codex_get_concurrency_status`
- `codex_get_worker_command_status`

`codex_get_operation_status` also returns `queueState`, `workerState`,
`slotState`, and `resourceLockState`. A running turn has
`slotState.claimed=true` and a `slotClaim` with the worker id, slot type, and
claim time. `codex_get_queue_status` separates queued work from running turn
operations, auxiliary operations, active turn slots, and lock conflicts.

When a workflow is waiting for capacity, `codex_get_workflow_status` mirrors the
nested operation queue state in `workflowOperationQueueState`. Use
`nextRecommendedAction="wait_for_worker_slot"` for slot pressure and
`nextRecommendedAction="wait_for_resource_lock"` for write lock conflicts. Do
not create another operation for the same work while either action is returned.

## First setup

The admin helper can generate a fuller client config, install hooks, and run a
protocol smoke:

```powershell
codex-control-plane-mcp-admin init --state-db .\state\codex-mcp-state.sqlite3 --projects-root C:\Users\you\Projects
```

The command prints a JSON block you can copy into an MCP client config. It does
not print secrets or private prompts.

You can also install only the Codex hooks:

```powershell
codex-control-plane-mcp-hooks install --state-db .\state\codex-mcp-state.sqlite3
codex-control-plane-mcp-hooks status
codex-control-plane-mcp-hooks doctor
```

The installer backs up `~/.codex/hooks.json`, merges its handlers with your
existing hooks, stores `stateDb` as an absolute path, and writes prompts, visible
agent progress text, final answers, and turn status into the MCP state DB. Tool
calls and command outputs are not recorded by default. Restart Codex after
installing or changing hooks.

For turns launched through `codex-app-server`, the server mirrors the accepted
prompt, visible assistant messages, and turn status into the same SQLite
history. That keeps search and status reads useful even when app-server does not
execute user hooks itself.

## Main workflows

Submit a durable task:

```text
codex_submit_task
  -> operationId
codex_get_operation_status(operationId)
  -> queued / running / waiting_for_approval / completed / failed
```

Use the same `client_request_id` when a caller retries after a transport timeout.
The retry returns the existing operation instead of creating another turn.

Attach screenshots or other image evidence:

```text
codex_submit_task(
  operation_type="start_chat",
  message="Analyze this screen.",
  input_items=[
    {"type": "localImage", "path": ".\\screens\\error.png", "detail": "low"},
    {"type": "image", "url": "https://example.com/screenshot.png", "detail": "high"}
  ]
)
```

Image inputs are accepted only for operation types that start a new turn:
`start_chat`, `send_message`, `execute_plan`, and `fork_thread` with an initial
message. MCP sends the path or URL to `codex-app-server`, but operation status
and diagnostics return only safe metadata such as type, detail, size, extension,
and hashes. Binary image content, raw URLs, and full local image paths are not
stored in public status payloads.

Steer an active turn:

```text
codex_submit_task(operation_type="steer_turn", thread_id=..., expected_turn_id=..., message=...)
  -> operationId
codex_get_operation_status(operationId)
  -> follows the target turn until completed / failed / interrupted
```

Use `steer_turn` only while the target turn is active. For a completed thread,
use `send_message` instead.

Fork a thread:

```text
codex_submit_task(operation_type="fork_thread", source_thread_id=...)
  -> operationId
codex_get_operation_status(operationId)
  -> completed, threadId=<forkedThreadId>
```

Start work in the fork right away:

```text
codex_submit_task(operation_type="fork_thread", source_thread_id=..., message=...)
  -> operationId
codex_get_operation_status(operationId)
  -> follows the first turn in the forked thread
```

Use `client_request_id` for retry-safe fork requests. Without it, each call is
treated as a new fork request. `threadId` in operation status is the forked
thread; the source thread is reported in `forkState.sourceThreadId`.

Manage thread lifecycle:

```text
codex_archive_thread(thread_id)
  -> completed
codex_unarchive_thread(thread_id)
  -> completed
codex_start_thread_compaction(thread_id)
  -> actionId
codex_get_thread_compaction_status(actionId)
  -> running / completed / unknown_after_app_server_exit
```

Archive and unarchive are audit actions around app-server `thread/archive` and
`thread/unarchive`. They refuse to run while the thread has an active turn or a
pending interaction. Compaction uses its own lightweight `actionId` because
`thread/compact/start` is asynchronous. Public `thread/delete` is intentionally
not exposed.

Ask for a structured final report:

```text
codex_submit_task(operation_type="start_chat", message=..., output_schema={...})
codex_approve_plan(workflowId, output_schema={...})
  -> operationId / executionOperationId
codex_get_operation_status(operationId)
codex_get_workflow_status(workflowId)
  -> finalReport.text + finalReport.structured
```

`output_schema` is passed to app-server `turn/start` and is tracked by a schema
hash in status output. Object schemas must use the strict form required by Codex:
set `additionalProperties` to `false`. MCP stores the final assistant message
as readable text, then parses JSON object output into `finalReport.structured`
when Codex returns valid JSON. Plain text still works and stays available in
`finalReport.text`.

MCP does not extract hidden chain-of-thought and does not store raw tool
payloads or command output in final reports.

Drive Plan Mode:

```text
codex_start_plan_workflow
  -> workflowId
codex_get_workflow_status(workflowId)
  -> wait_plan / review_plan / execute_plan
codex_approve_plan(workflowId)
  -> executionOperationId
codex_get_workflow_status(workflowId)
  -> finalReport
```

Plan Mode has a runtime floor. The public default write policy is still
`read-only` and `on-request`, but Plan Mode needs a writable workspace on
Windows. If the caller or server default resolves to `read-only`, MCP sends
`workspace-write` to `codex-app-server` and returns `requestedSandbox`,
`effectiveSandbox`, and `runtimePolicyAdjusted` in workflow and operation
status.

Mirror a workflow goal into Codex Desktop when the client has one:

```text
codex_start_plan_workflow(goal="Review the migration plan", goal_completion_action="clear")
codex_get_workflow_status(workflowId, refresh_live_goal=true)
  -> threadGoal.syncState + threadGoal.currentGoal
```

MCP writes a thread goal only when the client passes `goal`. Managed goals use
`clear` after completion by default. Use `set_complete` or `leave` when the goal
should remain visible after the workflow ends. Normal workflow polling is
passive; use `refresh_live_goal=true` only when you want MCP to call live
app-server goal methods.

Run a Codex code review:

```text
codex_start_review_workflow(thread_id=..., target_type="base_branch", base_branch="main")
  -> workflowId
codex_get_workflow_status(workflowId)
  -> wait_review / read_review_report
```

Or let MCP create a service thread for a local checkout:

```text
codex_start_review_workflow(cwd=..., target_type="uncommitted_changes")
  -> workflowId
codex_get_workflow_status(workflowId)
  -> reviewThreadId + reviewTurnId + finalReport
```

Review workflows do not write files by themselves. They run inside the selected
Codex sandbox and approval policy. Use `client_request_id` when a caller may
retry the start request after a transport timeout.

Handle approvals and questions:

```text
codex_list_pending_interactions
codex_answer_pending_interaction
```

Start diagnostics with:

```text
codex_get_runtime_capabilities
codex_health_summary
codex_collect_diagnostics
codex_analyze_issue
codex_repair_issue
```

Repair actions default to `dry_run=true`.

Status and diagnostic tools also return `agentGuidance` and
`agentGuidanceText` when MCP sees a blocker, failed state, stale run, pending
interaction, duplicate prompt, auth problem, rate limit, or unsafe recovery
loop. Agents should follow `agentGuidance.instructions` before deciding to
retry or stop. If `agentGuidance.loopGuard.allowed=false`, stop automatic
recovery, collect diagnostics, and ask a human. Do not create a new
`client_request_id` after a timeout unless the guidance explicitly says to start
a replacement workflow.

For a broken Plan Mode workflow, use
`retry_workflow_with_runtime_policy`. It creates a new workflow with the selected
sandbox and approval policy, links it to the old workflow through
`workflowRetryState`, and does not revive the old terminal turn.

`codex_health_summary` is about current readiness by default. Old stale or
orphaned rows are reported in `historicalDebt`, but they do not make fresh
orchestration look broken when the worker, queue, and app-server are currently
healthy. Use targeted cleanup for that debt instead of blocking new work.

Status payloads now separate freshness signals:

- `operationRowAgeSeconds`: age of the durable operation row;
- `turnFreshness.lastProgressAgeSeconds`: age of the last turn progress event;
- `workerFreshness.heartbeatAgeSeconds`: age of the worker heartbeat;
- `stalenessMeaning="operation_row_age"` for the compatibility
  `stalenessSeconds` field.

Public status payloads are agent-safe. Operation and workflow status return
`requestSummary` instead of raw `request`; it contains ids, runtime policy,
scheduling intent, input item state, output schema hash, resource keys, and
text hashes. It does not include the full prompt, full instructions, raw title,
raw image URL/path, exact token counts, raw command output, or private paths.
Use your own stored task text plus `requestSummary.*.sha256` for correlation.

`codex_get_queue_status` only recommends `wait_for_worker_slot` when there is
actual queued work blocked by slots. If there are running turns but
`queueSummary.queued == 0`, the queue action is `none`.

## Runtime capabilities

Use `codex_get_runtime_capabilities` before orchestration or after reconnect. It
starts the MCP-owned app-server if needed, calls short best-effort inventory
methods, and returns a cached snapshot for five minutes.

In `client` mode, the client process does not start its own app-server for live
inventory. It returns a passive worker-managed snapshot when one exists. With
`refresh=true`, it queues a worker command and returns `refreshCommandId`; poll
`codex_get_worker_command_status` to read the refreshed inventory.

The response includes:

- model count, default model, hidden flags, input modalities, reasoning efforts, and service tier count;
- permission profiles by `id` and `description`;
- Windows sandbox readiness;
- provider capabilities for web search, image generation, and namespace tools;
- hook and skill counts without raw hook commands or absolute skill paths;
- redacted account status, coarse usage bands, and operational rate-limit state;
- supported app-server schema methods with a compact source, version, and hash.

Account inventory is safe to show to an orchestrator. It reports whether Codex
is authenticated, the account and plan type, whether an email exists, whether
usage data is available, and whether a rate limit or credits issue is visible.
It does not return raw email, account identifiers, credit balances, spend
limits, exact spend used, daily usage buckets, or exact token counts.

If one inventory method times out or fails, the tool still returns `ok=true`
with `runtimeCapabilities.status="partial"` and a machine-readable warning in
`methodResults`. Set `refresh=true` to bypass the cache. `codex_health_summary`
shows a small