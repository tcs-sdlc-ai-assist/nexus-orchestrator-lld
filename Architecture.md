# Nexus Autonomous Developer Agent — Orchestrator Architecture

**Version:** 2.0
**Date:** 2026-04-22
**Status:** Design — superseeds Section 2 of `AUTONOMOUS_DEV_AGENT_TEAM_PLAN.md`
**Inspiration:** Claude Code (main-agent + Task sub-agents, file-memory, per-tool HITL)
**Core Stack:** LangGraph (outer graph + checkpointing), ReAct loop (inside orchestrator node), Pydantic (typed contracts), `filelock` (registry), PostgresSaver (multi-worker state)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Design Principles](#2-design-principles)
3. [System Topology](#3-system-topology)
4. [Component Layers](#4-component-layers)
5. [Orchestrator ReAct Loop](#5-orchestrator-react-loop)
6. [Worker Pool — Bounded Concurrency](#6-worker-pool--bounded-concurrency)
7. [Tool Catalog](#7-tool-catalog)
8. [Pydantic Model Catalog](#8-pydantic-model-catalog)
9. [Artifact Registry + Locking](#9-artifact-registry--locking)
10. [Memory Architecture](#10-memory-architecture)
11. [Core Flows](#11-core-flows)
12. [HITL Gates](#12-hitl-gates)
13. [LangGraph Integration](#13-langgraph-integration)
14. [State Schema](#14-state-schema)
15. [Concurrency, Backpressure, Timeouts](#15-concurrency-backpressure-timeouts)
16. [Error Handling Tiers](#16-error-handling-tiers)
17. [Observability](#17-observability)
18. [Deployment: Single → Multi-Worker](#18-deployment-single--multi-worker)
19. [Security Posture](#19-security-posture)
20. [Open Questions](#20-open-questions)

---

## 1. Executive Summary

### Problem

Build an autonomous developer agent that ingests heterogeneous artifacts (raw PRD, enriched PRD, multiple HLDs, multiple LLDs, user stories), analyzes them in parallel, synthesizes a master implementation plan, and executes the plan by generating/modifying code — all with Claude Code-grade per-tool HITL, resumable state, and bounded concurrency.

### Core Design

- **Single orchestrator ReAct loop** wrapped inside one LangGraph node.
- **Worker pool with fixed capacity (default 5)**. Orchestrator spawns workers via tool calls. Pool queues overflow.
- **Orchestrator autonomy:** LLM decides per-task whether to dispatch a worker or perform the work itself using its own file tools.
- **Workers are isolated ReAct loops** with restricted toolsets per phase (analysis vs. execution).
- **File-based context + memory** (`.nexus/`) — every durable handoff lives on disk, not in LangGraph state.
- **Registry-coordinated artifact claiming** with OS-level file locks.
- **LangGraph checkpointer** (Postgres in prod) = resumable thread state, per-tool HITL via `interrupt_before`.

### What Changed vs. Team Plan v1.0

| Team Plan v1.0 | v2.0 |
|---|---|
| `execute_parallel_workers()` using `asyncio.wait(FIRST_COMPLETED)` inside a node | Bounded worker pool with queue; orchestrator owns dispatch via tool calls |
| Send API fan-out | **Removed.** Pool tool replaces it. Orchestrator stays in control. |
| Single phase (blueprint → code) | **Three phases:** artifact analysis → plan synthesis → execution |
| Blueprint-only HITL | Plan-review HITL + per-mutation HITL |
| Orchestrator = dispatcher only | Orchestrator = first-class worker too; can read, write, search, edit directly |
| MemorySaver implicit | PostgresSaver-ready from day one |

---

## 2. Design Principles

1. **Orchestrator-in-control.** The LLM decides concurrency. No automatic fan-out. Dispatch is a tool call, not a graph construct.
2. **Bounded pool.** Never more than N workers running. Overflow queues, not parallelizes.
3. **Orchestrator works too.** For small or cross-cutting tasks, the orchestrator uses its own tools instead of spawning. One-file edits, quick reads, progress updates — no worker needed.
4. **Disk is the bus.** State that survives restarts lives on disk (`.nexus/`). LangGraph state holds references, not payloads.
5. **Typed handoffs.** Every worker → orchestrator handoff validates against a Pydantic schema. No free-form JSON.
6. **Capability, not policy.** Analysis workers literally do not have `edit_file` bound. ACL enforced at tool-binding time.
7. **Per-tool approval.** Mutation tools sit behind `interrupt_before`. Read tools do not.
8. **Explicit exploration.** Nothing injected implicitly. The agent reads files when it needs them.
9. **Resumable by default.** Every node boundary = checkpoint. Crash resumable by `thread_id`.
10. **Observable.** Every tool call, decision, and worker lifecycle event emits a stream event.

---

## 3. System Topology

```
┌───────────────────────────────────────────────────────────────────────┐
│  FastAPI / uvicorn (single → multi-worker)                            │
│  POST /dev-run       → start thread_id                                │
│  POST /dev-run/resume → resume from interrupt                         │
│  GET  /dev-run/state  → fetch checkpoint                              │
└──────────────────────────────┬────────────────────────────────────────┘
                               │
┌──────────────────────────────▼────────────────────────────────────────┐
│  LangGraph compiled graph (PostgresSaver checkpointer)                │
│                                                                       │
│  setup_workspace ──► orchestrator_node ──► hitl_plan_gate ──► ...     │
│                          ▲   │                                        │
│                          │   ▼                                        │
│                       tool_node (routed by conditional edge)          │
│                          │                                            │
│                          ├─► nav_tools       (no interrupt)           │
│                          ├─► scratchpad_tools (no interrupt)          │
│                          ├─► memory_tools     (no interrupt)          │
│                          ├─► registry_tools   (no interrupt)          │
│                          ├─► pool_tools       (no interrupt)          │
│                          ├─► mutation_tools   (interrupt_before ✓)    │
│                          └─► hitl_tools       (request_human_review)  │
└──────────────────────────────┬────────────────────────────────────────┘
                               │
┌──────────────────────────────▼────────────────────────────────────────┐
│  Worker Pool (asyncio TaskGroup, 5 slots, initialized per run)        │
│                                                                       │
│  Queue[WorkerJobSpec] ──► [Slot 1..5] ──► WorkerRunner (own ReAct)    │
│                                                      │                │
│                                                      ▼                │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │  Worker ReAct loop (per job)                                    │  │
│  │  bind_tools(phase-specific ACL)                                 │  │
│  │  → LLM → tool_call → execute → append → loop                    │  │
│  │  → structured scratchpad write → release claims                 │  │
│  └────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬────────────────────────────────────────┘
                               │
┌──────────────────────────────▼────────────────────────────────────────┐
│  Disk Layer (.nexus/)                                                 │
│  registry/artifacts.json   (filelock-protected)                       │
│  scratch/analysis/*.json   (worker findings, Pydantic-validated)      │
│  scratch/execution/*.json                                             │
│  memory/progress_full.md  progress_summary.md  session_log.jsonl      │
│  memory/master_plan.json  error_registry.json  decisions/*.md         │
│  workspace/               (source artifacts + generated code)         │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 4. Component Layers

### L0 — Contracts
Pydantic schemas for every cross-component payload (registry entries, worker specs, findings, plan DAG, tool results). See Section 8.

### L1 — Core Utilities
Pure Python, zero LLM/LangGraph dependencies. Independently testable.

- `FileContextManager` — line-numbered reads, SHA-256, token counting, truncation with pagination warnings.
- `FileLockManager` — file-level async locks (per-file write coordination during mutation).
- `CodeMutatorPipeline` — 6-gate atomic write (sanitize → fuzzy match → hash check → format → atomic swap).
- `ArtifactRegistry` — `filelock`-backed JSON registry (claim/release/list/expire).
- `ScratchpadStore` — typed worker findings read/write.
- `AgentMemory` — `progress_full.md`, `progress_summary.md`, `session_log.jsonl`, `error_registry.json`, `decisions/`.

### L2 — Tool Registry
Grouped tool bundles with per-group ACL. Each group emits `StructuredTool` instances with Pydantic arg schemas.

- `nav_tools` — `list_directory`, `glob_search`, `grep_search`, `read_file_chunk`.
- `scratchpad_tools` — `write_scratchpad`, `read_scratchpad`, `list_scratchpads`.
- `memory_tools` — `append_progress_full`, `write_progress_summary`, `log_decision`, `lookup_known_error`.
- `registry_tools` — `claim_artifact`, `release_artifact`, `list_free_artifacts`, `list_claimed_artifacts`.
- `pool_tools` — `spawn_worker`, `check_worker_status`, `await_workers`, `list_workers`, `cancel_worker`.
- `mutation_tools` — `create_file`, `edit_file`, `run_terminal`.
- `hitl_tools` — `request_human_review`, `write_master_plan`.

### L3 — ReAct Engines
Orchestrator and worker share the same engine, differ only in tool bindings.

- `OrchestratorAgent` — ReAct loop with full tool surface, manages memory, owns worker pool lifecycle.
- `WorkerAgent` — ReAct loop with restricted tools per role; terminates when structured scratchpad written.

### L4 — LangGraph Integration
- `DevPhaseState` — typed state.
- `dev_phase_builder` — graph assembly with `interrupt_before` on mutation + plan gate.
- `hitl_router` — conditional edge (no `execution_complete` flag needed — end detected by empty `tool_calls`).

### L5 — Worker Pool
- `WorkerPool` — fixed-capacity async task group with FIFO queue, per-job status, cancellation, timeout.

### L6 — API Surface
- FastAPI endpoints for run/resume/state.
- SSE streaming endpoint for event emission.
- Postgres connection pool for `PostgresSaver`.

---

## 5. Orchestrator ReAct Loop

### Responsibilities

1. Initialize `.nexus/` layout + artifact registry.
2. Read enriched inputs, group artifacts for analysis.
3. Decide per-task: **spawn worker** or **do it myself**.
4. Maintain `progress_summary.md` (self-written, capped).
5. Synthesize `master_plan.json` from analysis findings.
6. Request human review of plan (HITL gate 1).
7. Dispatch execution waves respecting DAG dependencies.
8. Merge execution findings, handle errors, retry or escalate.
9. Finalize: commit workspace, write final report.

### Loop Shape

```
while True:
    inject_context = [
        system_prompt,
        read(.nexus/memory/progress_summary.md),
        read(.nexus/registry/artifacts.json) summary,
        pool.snapshot()  # free slots, queue depth, running jobs
    ]
    ai_message = llm.ainvoke(messages + inject_context)
    messages.append(ai_message)

    if not ai_message.tool_calls:
        return ai_message.content  # loop ends naturally

    # Route to ToolNode (via conditional edge outside this loop)
    tool_results = execute_tools(ai_message.tool_calls)
    messages.extend(tool_results)
```

**Implementation note:** the "loop" is expressed as LangGraph nodes (`orchestrator_agent` → `tool_node` → `orchestrator_agent`), not a Python while-loop. Each cycle = one checkpoint. Mid-loop crash is resumable.

### Decision Heuristics (prompt-encoded, not hardcoded)

Orchestrator system prompt teaches it:

- **Spawn worker** when: multiple artifacts need deep analysis, task exceeds ~2K token read budget, task is independent of other in-flight work, or execution DAG task has own sandbox scope.
- **Do it myself** when: single-file edit, small cross-cutting change, progress/memory update, registry query, plan synthesis, error diagnosis.
- **Queue don't block** when: pool at capacity and task is parallelizable — call `spawn_worker` and continue other work, then `await_workers` later.
- **Await selectively** — use `await_workers([job_ids], timeout)` on specific jobs, not whole pool.

### Iteration Budget

- Soft cap: 80 iterations per phase (analysis, synthesis, execution each tracked separately).
- Hard cap: 200 total iterations per run.
- Exceeds cap → emit Tier-3 error, finalize with partial results, HITL.

---

## 6. Worker Pool — Bounded Concurrency

### Why Not `Send` API

`Send` fans out one worker per invocation; concurrency control is implicit and per-compilation. User requires:

- Hard cap of 5 concurrent workers regardless of tasks queued.
- Orchestrator free to decide timing of dispatch vs. self-execution.
- Workers queueable independently across orchestrator iterations.
- Orchestrator able to continue its own work while pool drains.

Bounded tool-driven pool gives that. `Send` does not.

### Pool Mechanics

```
WorkerPool:
  capacity:    int = 5
  queue:       asyncio.Queue[WorkerJobSpec]
  slots:       list[asyncio.Task]          (5 long-running consumers)
  statuses:    dict[job_id, WorkerJobStatus]  (disk-backed for resumability)
  cancellers:  dict[job_id, asyncio.Event]

  spawn(spec) -> job_id                (non-blocking; enqueues)
  await_job(job_id, timeout) -> result (blocking; awaits one)
  await_all(job_ids, timeout) -> list  (blocking; awaits set)
  snapshot() -> PoolSnapshot           (free slots, queue depth, running)
  cancel(job_id) -> bool               (sets cancellation event)
```

Slots run:

```
async def slot_runner(slot_id):
    while True:
        spec = await queue.get()
        status.set(spec.job_id, RUNNING)
        try:
            result = await WorkerAgent(spec).run_with_timeout(spec.timeout_s)
            status.set(spec.job_id, DONE, result)
        except Cancelled:
            status.set(spec.job_id, CANCELLED)
        except Exception as e:
            status.set(spec.job_id, FAILED, error=e)
        emit_event(...)
        queue.task_done()
```

### Status States

```
QUEUED → RUNNING → {DONE, FAILED, CANCELLED, TIMED_OUT}
```

### Disk-Backed Status

Pool writes each status transition to `.nexus/registry/worker_jobs.json` under filelock. On orchestrator resume after crash, pool rehydrates `statuses` from disk; in-flight RUNNING jobs whose workers died are marked STALE and re-queued (idempotent by `job_id`).

### Lifecycle

- Pool created when orchestrator node entered for first time per run.
- Slots persist across orchestrator ReAct iterations within same run.
- On graph completion or `interrupt`: slots suspended (queue preserved), resumed on `ainvoke(None, ...)`.
- On run termination: slots drained gracefully, any running worker given `graceful_shutdown_s` to finish.

### Concurrency Guard Rails

- Queue depth cap: 50. Overflow → `spawn_worker` tool returns `ERROR: pool_saturated`. Orchestrator must await before queuing more.
- Per-job timeout: default 600s, per-spec overridable up to 1800s.
- Global pool timeout: 3600s total slot-time; orchestrator must finalize.

---

## 7. Tool Catalog

### Tool Group Matrix

| Tool | Orchestrator | Analysis Worker | Execution Worker |
|---|:-:|:-:|:-:|
| `list_directory` | ✓ | ✓ | ✓ |
| `glob_search` | ✓ | ✓ | ✓ |
| `grep_search` | ✓ | ✓ | ✓ |
| `read_file_chunk` | ✓ | ✓ | ✓ |
| `write_scratchpad` | ✓ | ✓ | ✓ |
| `read_scratchpad` | ✓ | ✓ | ✓ |
| `list_scratchpads` | ✓ | ✓ | ✓ |
| `append_progress_full` | ✓ | ✗ | ✗ |
| `write_progress_summary` | ✓ | ✗ | ✗ |
| `log_decision` | ✓ | ✓ | ✓ |
| `lookup_known_error` | ✓ | ✓ | ✓ |
| `claim_artifact` | ✓ | ✓ | ✓ |
| `release_artifact` | ✓ | ✓ | ✓ |
| `list_free_artifacts` | ✓ | ✓ | ✓ |
| `list_claimed_artifacts` | ✓ | ✓ | ✓ |
| `spawn_worker` | ✓ | ✗ | ✗ |
| `check_worker_status` | ✓ | ✗ | ✗ |
| `await_workers` | ✓ | ✗ | ✗ |
| `list_workers` | ✓ | ✗ | ✗ |
| `cancel_worker` | ✓ | ✗ | ✗ |
| `create_file` | ✓ (HITL) | ✗ | ✓ (HITL) |
| `edit_file` | ✓ (HITL) | ✗ | ✓ (HITL) |
| `run_terminal` | ✓ (HITL) | ✗ | ✓ (HITL) |
| `request_human_review` | ✓ | ✗ | ✗ |
| `write_master_plan` | ✓ | ✗ | ✗ |

Analysis workers are **read-only** by capability. They cannot mutate code or filesystem state beyond their own scratchpad.

### Tool Contracts (abbreviated)

Each tool defines:
- `name` — LLM-visible name.
- `description` — includes usage guidance + failure modes (e.g., "replaces ONLY FIRST occurrence").
- `args_schema` — Pydantic BaseModel with `Field(..., description=...)` for every arg.
- `handler` — async function returning Pydantic result model.
- `requires_interrupt` — bool; gates mutation + HITL tools.

### Key Tool Behaviors

**`spawn_worker(worker_spec: WorkerJobSpec) → WorkerJobAck`**
- Validates spec, assigns `job_id`, enqueues.
- Returns `{job_id, queued_position, estimated_start_eta}` immediately.
- Never blocks. Orchestrator must `await_workers` later if needs result.

**`await_workers(job_ids: list[str], timeout_s: int = 600) → AwaitResult`**
- Blocks until all job_ids reach terminal state or timeout.
- Returns `{completed: [...], failed: [...], still_running: [...], findings_paths: [...]}`.
- Partial timeouts: returned still-running jobs can be polled or cancelled.

**`list_workers() → PoolSnapshot`**
- Returns `{capacity, running_count, queue_depth, free_slots, jobs: [{id, status, worker_type, age_s}]}`.
- Called frequently; no side effects.

**`claim_artifact(artifact_path: str, worker_id: str, ttl_s: int = 600) → ClaimResult`**
- Atomic check-and-set via `filelock`.
- Auto-expires stale claims past TTL.
- Returns success + claim token, or failure with reason (`already_locked_by: X`).

**`write_master_plan(plan: MasterPlan) → PlanWriteResult`**
- Validates `MasterPlan` Pydantic schema (DAG integrity check, cycle detection).
- Persists to `.nexus/memory/master_plan.json`.
- Subsequent execution dispatch reads this.

**`request_human_review(payload: HumanReviewRequest) → ReviewResult`**
- Raises `GraphInterrupt` carrying payload to UI.
- Resumable via `ainvoke(Command(resume=ReviewDecision(...)), ...)`.

---

## 8. Pydantic Model Catalog

### Registry

```
ArtifactRegistryEntry:
  artifact_path: str
  artifact_type: Literal["raw_prd", "enriched_prd", "hld", "lld", "user_story", "supporting"]
  status: Literal["free", "locked", "done", "failed"]
  locked_by: Optional[str]         # worker_id or "orchestrator"
  locked_at: Optional[datetime]
  ttl_s: int = 600
  findings_path: Optional[str]
  completed_at: Optional[datetime]
  hash_sha256: str
```

### Worker Job

```
WorkerJobSpec (discriminated union by worker_type):
  job_id: str                       # UUID4
  worker_type: Literal["analysis", "execution"]
  assigned_artifacts: list[str]     # or execution task_ids
  scratchpad_path: str
  timeout_s: int = 600
  context_budget_tokens: int = 8000
  parent_thread_id: str
  created_at: datetime
  priority: int = 0

AnalysisJobSpec(WorkerJobSpec):
  worker_type: "analysis"
  analysis_focus: Literal["structural", "requirements", "dependency", "holistic"]

ExecutionJobSpec(WorkerJobSpec):
  worker_type: "execution"
  task_id: str                      # reference to MasterPlan.tasks
  target_files: list[str]
  depends_on: list[str] = []
```

### Worker Status

```
WorkerJobStatus:
  job_id: str
  status: Literal["queued", "running", "done", "failed", "cancelled", "timed_out", "stale"]
  started_at: Optional[datetime]
  finished_at: Optional[datetime]
  findings_path: Optional[str]
  error_summary: Optional[str]
  retry_count: int = 0
```

### Findings

```
AnalysisFindings:
  worker_id: str
  job_id: str
  artifacts_analyzed: list[str]
  artifact_type: str
  key_entities: list[str]
  key_requirements: list[RequirementRef]
  identified_dependencies: list[DependencyEdge]
  open_questions: list[str]
  suggested_tasks: list[TaskProposal]
  confidence: float                 # 0..1
  token_budget_used: int

ExecutionResult:
  worker_id: str
  job_id: str
  task_id: str
  files_created: list[str]
  files_modified: list[FileModification]  # path + hash_before + hash_after
  tests_added: list[str]
  terminal_invocations: list[TerminalInvocation]
  blockers: list[str]
  status: Literal["completed", "partial", "blocked"]
  notes: str
```

### Plan

```
MasterPlan:
  plan_id: str
  created_at: datetime
  input_artifacts: list[str]
  tasks: list[TaskNode]
  edges: list[DependencyEdge]       # DAG
  human_notes: Optional[str]
  approved_at: Optional[datetime]
  approved_by: Optional[str]

TaskNode:
  task_id: str
  title: str
  description: str
  target_files: list[str]
  depends_on: list[str]
  estimated_complexity: int         # 1..5
  assigned_worker_type: Literal["analysis", "execution", "orchestrator_self"]
  acceptance_criteria: list[str]

DependencyEdge:
  from_task: str
  to_task: str
  edge_type: Literal["blocks", "informs", "co_changes"]
```

### Orchestrator State

```
DevPhaseState (TypedDict extension):
  run_id: str
  thread_id: str
  workspace_path: str
  nexus_path: str
  messages: Annotated[list, trim_messages(max_tokens=6000, strategy="last")]
  phase: Literal["setup", "analysis", "synthesis", "plan_review", "execution", "finalize"]
  plan_approved: Optional[bool]
  iterations_by_phase: dict[str, int]
  errors: list[ErrorEntry]
  final_report_path: Optional[str]
```

No `execution_complete` flag. Loop ends naturally when LLM returns without `tool_calls`.

### Tool Results

Every tool returns a Pydantic model serialized to JSON and injected as `ToolMessage.content`. Never raw strings. Schemas ensure the LLM sees structured error + success payloads.

```
ToolResult (base):
  success: bool
  timestamp: datetime
  error_code: Optional[str]
  error_message: Optional[str]
  hint: Optional[str]               # remediation suggestion for LLM

FileReadResult(ToolResult):
  path: str
  content: str                      # line-numbered
  total_lines: int
  hash_sha256: str
  tokens_used: int
  truncated: bool
  next_offset: Optional[int]

... (one per tool)
```

### Memory

```
ProgressEntry:
  timestamp: datetime
  phase: str
  iteration: int
  action: str
  result_summary: str               # 200 char max

DecisionLogEntry:
  timestamp: datetime
  decision_id: str
  phase: str
  rationale: str
  alternatives_considered: list[str]
  tool_call_ref: Optional[str]

ErrorEntry:
  timestamp: datetime
  tier: Literal[1, 2, 3]
  component: str
  error_message: str
  remediation_applied: Optional[str]
  resolved: bool
```

---

## 9. Artifact Registry + Locking

### Why Disk-Based

- Survives orchestrator crash.
- Sharable across uvicorn workers once multi-worker (single JSON on shared FS, or swap to Postgres table).
- No distributed lock manager dependency for single-node deployment.
- Upgrade path = swap registry backend, keep tool contract.

### Locking Library

`filelock` (pure Python, cross-platform, OS advisory lock). Lock file at `.nexus/registry/artifacts.json.lock`. All read/modify/write cycles inside `with FileLock(...):`.

### Lock Granularity

- **Registry-level lock** for every claim/release (milliseconds held).
- **File-level locks** (plan v1.0's `FileLockManager`) used during mutation-phase file writes, orthogonal to registry.
- **TTL auto-expiry** on registry entries to recover from dead workers.

### Registry Operations (summary)

- `claim(path, worker_id, ttl)` — atomic; fails if already locked and TTL unexpired.
- `release(path, worker_id, findings_path)` — lock-holder only; status → done.
- `abandon(path, worker_id, reason)` — lock-holder returns artifact to free state with error note.
- `list_free()` — snapshot of unclaimed artifacts.
- `list_claimed_by(worker_id)` — workers can introspect own claims.
- `prune_expired()` — cron-style sweep called at orchestrator loop start.

### Upgrade to Postgres

```
Table artifact_registry (
  artifact_path text primary key,
  artifact_type text,
  status text,
  locked_by text,
  locked_at timestamptz,
  ttl_s int,
  findings_path text,
  hash_sha256 text
)

-- claim:
BEGIN;
SELECT * FROM artifact_registry
  WHERE artifact_path = $1 FOR UPDATE;
-- if free or expired, UPDATE status='locked' ...
COMMIT;
```

Multi-worker safe. Same tool interface.

---

## 10. Memory Architecture

### Three Tiers

| Tier | Lifetime | Storage | Injected? |
|---|---|---|---|
| LangGraph state | one run (checkpointed) | Postgres | yes (messages trimmed) |
| `.nexus/memory/` | one run + post-mortem | disk | selectively (summary only) |
| Error registry | cross-run | disk (or Postgres) | on-demand lookup |

### Files

```
.nexus/
  registry/
    artifacts.json
    artifacts.json.lock
    worker_jobs.json
    worker_jobs.json.lock
  scratch/
    analysis/
      worker_{job_id}.json         # AnalysisFindings
    execution/
      worker_{job_id}.json         # ExecutionResult
  memory/
    progress_full.md               # append-only chronology
    progress_summary.md            # LLM-maintained, capped ~500 tokens
    session_log.jsonl              # machine audit trail (one JSON per line)
    master_plan.json               # synthesized plan
    error_registry.json            # known errors + resolutions
  decisions/
    decision_{timestamp}_{id}.md   # long-form rationale
  workspace/
    requirements/
      raw_prd.md
      enriched_prd.md
    hld/
      hld_*.md
    lld/
      lld_*.md
    user_stories/
      story_*.md
    generated_code/
      (all project source)
```

### progress_summary.md Contract

- Written by orchestrator via `write_progress_summary` tool.
- Tool validates size: must be ≤ 600 tokens.
- Tool rejects if LLM tries to cram full history — forces summarization discipline.
- Injected into orchestrator system prompt every iteration.

### progress_full.md Contract

- Append-only.
- `append_progress_full(entry: ProgressEntry)` — one entry per tool call + per phase transition.
- Never injected into prompt. Available via `read_file_chunk` if agent chooses to consult.

### session_log.jsonl Contract

- One JSON object per line, one line per event.
- Event types: `tool_call`, `tool_result`, `worker_spawn`, `worker_complete`, `phase_transition`, `hitl_request`, `hitl_resume`, `error`.
- Machine-readable for post-hoc analysis, no LLM consumption.

### Error Registry Contract

- Keyed by `error_fingerprint` (hash of error_code + component).
- Value: `{first_seen, last_seen, occurrences, known_resolutions: [...]}`.
- Orchestrator consults via `lookup_known_error` on Tier-2 errors.
- Tier-1 auto-retry (tenacity) does not update registry; Tier-2 resolutions do.

---

## 11. Core Flows

### 11.1 Workspace Setup

```
1. setup_workspace node entered
   ├─ validate DevPhaseState (non-None artifacts)
   ├─ materialize workspace/ from input state
   │    ├─ requirements/raw_prd.md, enriched_prd.md
   │    ├─ hld/*.md, lld/*.md
   │    └─ user_stories/*.md
   ├─ init .nexus/ skeleton
   ├─ build initial artifacts.json registry (all "free")
   ├─ compute SHA-256 per artifact
   └─ write workspace_ready event
2. Edge → orchestrator_node
```

### 11.2 Analysis Phase

```
orchestrator_node (iteration loop):
  Iter 1:  read registry → see N artifacts free
           decide: spawn 3 analysis workers on small artifacts, handle 2 myself
           tool_calls: [
             spawn_worker(AnalysisJobSpec(artifacts=[prd.md], focus=requirements)),
             spawn_worker(AnalysisJobSpec(artifacts=[hld_1.md, hld_2.md], focus=structural)),
             spawn_worker(AnalysisJobSpec(artifacts=[lld_1.md], focus=dependency)),
             claim_artifact(user_story_1.md, orchestrator),
             claim_artifact(user_story_2.md, orchestrator),
           ]

  Iter 2:  tool_results back. 3 job_ids queued.
           Read user_story_1 directly via read_file_chunk.
           Append observations to progress_full.

  Iter 3:  Read user_story_2. Release own claims with findings_path.
           Call await_workers([job_ids], timeout=600).
           [blocking — returns when all 3 worker findings ready]

  Iter 4:  await_workers returned. Read scratch/analysis/*.json via glob + read_file_chunk.
           Each AnalysisFindings validated on read.
           Update progress_summary with "analysis phase complete, N findings".
           Transition phase → synthesis.
```

Worker concurrency throughout: ≤ 5. Queued if more spawned.

### 11.3 Synthesis Phase

```
orchestrator_node continuing:
  Iter 5:  Read all AnalysisFindings.
           LLM synthesizes MasterPlan (DAG of execution tasks).
           tool_call: write_master_plan(MasterPlan(...))
             → Pydantic validation (cycle check, ref integrity)
             → persist to .nexus/memory/master_plan.json

  Iter 6:  tool_call: request_human_review(HumanReviewRequest(
             payload_type="master_plan",
             payload_path=".nexus/memory/master_plan.json",
             summary="N tasks, M dependencies, estimated K workers"
           ))
           → raises GraphInterrupt
           → graph pauses, state checkpointed
           → uvicorn returns interrupt payload to caller
```

### 11.4 Plan Review (HITL Gate 1)

```
External UI:
  displays master_plan.json
  human reviews
  returns ReviewDecision(approved=true|false, edits=[...], comments="...")

Caller resumes:
  ainvoke(Command(resume=decision), config={"thread_id": X})

If approved=false:
  orchestrator receives ReviewDecision in tool result
  applies edits (maybe spawn analysis workers for re-analysis)
  regenerate plan, re-request review

If approved=true:
  phase transitions to execution
```

### 11.5 Execution Phase

```
orchestrator_node:
  Iter N:  Read approved MasterPlan.
           Identify first wave (tasks with no deps or satisfied deps).
           For each task in wave:
             decide spawn vs. self
             if spawn:
               spawn_worker(ExecutionJobSpec(task_id=X, target_files=[...]))
             else:
               claim target files, call edit_file/create_file directly
                 → interrupt_before triggers per-tool HITL
                 → user approves or rejects
                 → resume

  await_workers on wave → all done → read execution scratchpads
  update master_plan.json with completed_tasks
  identify next wave → dispatch
  loop until DAG drained
```

### 11.6 Per-Mutation HITL (Gate 2)

```
Mutation tool called (by orchestrator or execution worker):
  ToolNode sees tool_call in mutation_tools group
  interrupt_before fires
  graph pauses; state checkpointed
  caller receives pending tool_calls payload

UI presents:
  - tool: edit_file
  - path: src/foo.py
  - diff: (computed from args)
  - justification: last AIMessage content
  - phase: execution
  - worker_id: (if worker)

Human approves -> resume with Command(resume={"approved": true})
  -> ToolNode executes mutation
  -> flows back to agent_node
  -> agent sees ToolMessage with result

Human rejects -> resume with Command(resume={"approved": false, "reason": "..."})
  -> ToolNode emits synthetic ToolMessage: "rejected by human: <reason>"
  -> agent reasons next step (retry differently, escalate, skip)
```

### 11.7 Error Recovery

```
Tool raises exception:
  Tier 1 (transient): tenacity wrapper retries 5x exp backoff
    -> success: proceed
    -> exhausted: escalate to Tier 2

  Tier 2 (correctable by agent):
    wrap as ToolResult(success=false, error_code=X, error_message=Y, hint=Z)
    return as ToolMessage.content
    agent sees, reasons, tries alternative
    if repeats 3x same fingerprint -> escalate to Tier 3

  Tier 3 (fatal):
    append to state.errors
    log to session_log.jsonl + error_registry.json
    if execution-phase: partial finalize with what is done
    if pre-plan-approval: terminate with "analysis failed"
    always: emit event for UI
```

### 11.8 Finalization

```
orchestrator_node final iteration (LLM returns no tool_calls):
  conditional edge -> finalize_node
    |- compile final report (tasks done, files changed, blockers)
    |- write .nexus/memory/final_report.md
    |- drain pool (graceful)
    |- release all still-claimed artifacts
    |- emit run_complete event
  -> END
```

---

## 12. HITL Gates

### Gate 1: Master Plan Review (mandatory)

- **When:** after synthesis, before execution dispatch.
- **How:** `request_human_review(payload_type="master_plan")` -> `GraphInterrupt`.
- **Payload:** full `master_plan.json` + summary.
- **Resume:** `Command(resume=ReviewDecision)`.
- **Behavior on reject:** orchestrator applies edits, possibly re-spawns analysis, regenerates plan.

### Gate 2: Per-Mutation Approval (always-on)

- **When:** before every `create_file`, `edit_file`, `run_terminal` call.
- **How:** `interrupt_before=["tool_node_mutating"]` on compiled graph.
- **Payload:** pending `tool_calls` from last AIMessage; app-layer computes diffs for display.
- **Resume:** `Command(resume={"approved": true|false, "reason": "..."})`.
- **Behavior on reject:** ToolNode generates synthetic rejection ToolMessage, agent reasons alternative.

### Gate 3: Error Escalation (on-demand)

- **When:** Tier-3 error requires human guidance.
- **How:** `request_human_review(payload_type="error_escalation")`.
- **Payload:** error context + recovery options.
- **Resume:** human decides abort or guide.

### Gate Configuration

Per-deployment via env var:

```
HITL_MODE = strict | standard | relaxed
  strict   -> all 3 gates + confirmation on every worker spawn
  standard -> gates 1+2 active (default for prod)
  relaxed  -> gate 1 only (dev/staging)
```

---

## 13. LangGraph Integration

### Graph Shape

```
Graph (StateGraph[DevPhaseState]):

  nodes:
    setup_workspace         (sync, no LLM)
    orchestrator_agent      (LLM + bind_tools, ReAct step)
    tool_node_safe          (nav, scratchpad, memory, registry, pool)
    tool_node_mutating      (mutation tools) [interrupt_before]
    tool_node_hitl          (request_human_review) [interrupt_before]
    finalize                (sync, cleanup + report)

  edges:
    START -> setup_workspace
    setup_workspace -> orchestrator_agent
    orchestrator_agent -> (cond) -> tool_node_safe | tool_node_mutating | tool_node_hitl | finalize
    tool_node_safe -> orchestrator_agent
    tool_node_mutating -> orchestrator_agent
    tool_node_hitl -> orchestrator_agent

  conditional_logic (on orchestrator_agent output):
    if last_ai_message.tool_calls is empty -> finalize
    elif any tool_call.name in mutation_tool_names -> tool_node_mutating
    elif any tool_call.name in hitl_tool_names -> tool_node_hitl
    else -> tool_node_safe

  interrupt_before: ["tool_node_mutating", "tool_node_hitl"]

  checkpointer: AsyncPostgresSaver (prod) | MemorySaver (dev)
```

### State Reducers

```
messages: Annotated[list[BaseMessage], add_messages_trimmed]
  where add_messages_trimmed applies:
    - standard add_messages (append ToolMessages, dedupe by id)
    - then trim_messages(max_tokens=6000, strategy="last", start_on="human")
  keeps most recent window; older history lives in progress_full.md on disk
```

### Checkpointer Setup

```
Dev:   MemorySaver()
Prod:  AsyncPostgresSaver(async_conn_pool)
       with psycopg_pool.AsyncConnectionPool(
         conninfo=POSTGRES_URL,
         min_size=2, max_size=10,
       )
Setup: await saver.setup()  # auto-creates checkpoint tables
```

### Thread Model

- `thread_id` = unique per dev-phase run. Format: `dev_run_{task_id}_{timestamp}`.
- Worker subthread IDs: `{parent_thread_id}_worker_{job_id}`.
- Worker checkpoints in same Postgres; can be partitioned by parent.

---

## 14. State Schema

```
DevPhaseState (TypedDict):
  # identity
  run_id: str
  thread_id: str
  task_id: str                      # upstream task reference

  # paths
  workspace_path: str
  nexus_path: str

  # messaging
  messages: Annotated[list, add_messages_trimmed]

  # phase tracking
  phase: Literal["setup", "analysis", "synthesis", "plan_review",
                 "execution", "finalize"]
  iterations_by_phase: dict[str, int]

  # plan
  plan_approved: Optional[bool]
  plan_revision: int                # increments on re-synthesis

  # errors
  errors: list[ErrorEntry]

  # output
  final_report_path: Optional[str]
```

### What Does NOT Go in State

- Worker pool object (runtime; reconstructed from disk on resume).
- Full findings payloads (stored in scratchpads on disk).
- Full plan JSON (stored in master_plan.json on disk).
- Raw artifact contents (read via tools when needed).

State stays small -> checkpoint size bounded -> Postgres efficient.

---

## 15. Concurrency, Backpressure, Timeouts

### Limits

| Limit | Default | Configurable |
|---|---|:-:|
| MAX_CONCURRENT_WORKERS | 5 | env |
| MAX_QUEUE_DEPTH | 50 | env |
| WORKER_TIMEOUT_S | 600 | env + per-spec |
| GLOBAL_POOL_TIMEOUT_S | 3600 | env |
| ORCHESTRATOR_MAX_ITERATIONS | 200 | env |
| PHASE_MAX_ITERATIONS | 80 | env |
| CONTEXT_BUDGET_TOKENS | 6000 | env |
| MESSAGE_WINDOW_TOKENS | 6000 | env |
| CLAIM_TTL_S | 600 | env |
| PROGRESS_SUMMARY_MAX_TOKENS | 600 | env |
| POSTGRES_POOL_MIN | 2 | env |
| POSTGRES_POOL_MAX | 10 | env |

### Backpressure

- Pool queue full -> `spawn_worker` returns `ERROR: pool_saturated, queue_depth=50`. Orchestrator must `await_workers` before queuing more.
- Registry claim fails -> orchestrator receives `{success: false, already_locked_by: X}`; reasons about alternatives.
- Message window exceeds budget -> reducer trims oldest; full history on disk.

### Graceful Degradation

- On pool saturation: orchestrator prompt explicitly encourages self-execution or await.
- On repeated worker failures: orchestrator switches to self-execution for remaining tasks.
- On Postgres connection loss: checkpointer retries via `psycopg_pool`; tool calls queue until reconnect.

---

## 16. Error Handling Tiers

| Tier | Examples | Handler | Agent visibility |
|---|---|---|---|
| **1 - Transient** | HTTP 429, 502, Postgres transient, timeouts | `tenacity` retry (5 x exp backoff) | No |
| **2 - Semantic** | FileNotFoundError, stale claim, hash mismatch, Pydantic validation fail, worker scratchpad invalid | ToolResult with success=false + hint; logged | Yes - agent reasons |
| **3 - Fatal** | Path traversal, whitelist violation, iteration cap, pool deadlock, unrecoverable Postgres | Append to state.errors, HITL Gate 3, or terminate | Pipeline halts |

### Retry Policies

- Tier 1: tenacity `stop_after_attempt(5)` + `wait_exponential(1, 60)`.
- Tier 2: agent-driven; repeated same-fingerprint failures -> Tier 3.
- Tier 3: no retry; human intervention or run termination.

### Poison Pill Detection

Worker that fails 3x with same error fingerprint marked `poison`. Its claimed artifacts released with note; not re-queued automatically. Orchestrator receives a signal to decide manual takeover or abandonment.

---

## 17. Observability

### Streaming Events

Every tool call, phase transition, worker lifecycle, HITL request emits via `stream_writer.create_emitter()`.

Event types:

```
phase_transition      {from, to, timestamp}
tool_call             {tool_name, args_summary, caller: orchestrator|worker_id}
tool_result           {tool_name, success, latency_ms, truncated_result}
worker_spawn          {job_id, worker_type, assigned_artifacts}
worker_lifecycle      {job_id, status, duration_s}
hitl_request          {gate, payload_summary}
hitl_resume           {gate, decision}
error                 {tier, component, message}
progress_summary_update
memory_write
```

### Metrics

- Histogram: tool call latency per tool_name.
- Counter: worker completions by status.
- Gauge: pool queue depth, running count.
- Counter: HITL rejections by tool_name (signals prompt quality).
- Counter: Tier 1/2/3 errors by component.

### Tracing

LangSmith integration via standard env vars. Every ReAct cycle = one trace. Sub-worker runs = child traces (parent_run_id linkage).

### Audit

`session_log.jsonl` immutable, grep-friendly. Shipped with `.nexus/` directory on final commit.

---

## 18. Deployment: Single -> Multi-Worker

### Single-Worker (Current)

```
uvicorn app:api --workers 1
Checkpointer: AsyncPostgresSaver (from day one)
Pool: in-process asyncio TaskGroup
Registry: filelock-protected JSON
```

### Multi-Worker (Next Month)

```
uvicorn app:api --workers 4
Checkpointer: AsyncPostgresSaver (unchanged - thread_id routes to checkpoint)
Pool: one per run, lives inside worker process that hosts that thread_id
Registry: swap JSON -> Postgres table with SELECT FOR UPDATE SKIP LOCKED
```

### Thread Affinity - Two Options

1. **Sticky routing** (simpler): caller passes `thread_id`, ingress routes to same uvicorn worker via consistent hash. Pool stays in-process. Zero code change.
2. **Stateless workers** (cleaner): pool state on disk or Postgres. Any uvicorn worker can resume any thread. Pool rehydrates from `worker_jobs.json` or a Postgres-backed queue (`pgmq`, `arq`, or advisory-lock polling).

Recommendation: start sticky routing. Migrate to stateless when horizontal scale demands it.

### Migration Checklist

When moving to multi-worker:

1. Swap registry JSON -> Postgres table.
2. Add ingress sticky routing OR adopt stateless pool.
3. Verify Postgres connection pooling (`psycopg_pool`) tuned.
4. Test interrupt/resume across workers.
5. Add distributed tracing headers.
6. Load-test pool saturation under concurrent orchestrator runs.

No orchestrator/worker ReAct logic changes. Interface-stable by design.

---

## 19. Security Posture

### Terminal Command Whitelist

```
Allowed: npm, npx, node, python, python3, pytest, pip, pip3,
         black, prettier, eslint, tsc, go, gofmt, cargo, make,
         mkdir, touch, tree
Blocked: cat, echo, ls, dir, curl, wget, rm, mv, cp, chmod,
         chown, sudo, su, bash, sh, zsh, powershell, cmd
```

### Path Traversal Defense

- All path inputs canonicalized via `Path(...).resolve()`.
- Rejected if not descendant of `workspace_root`.
- `..` segments in args rejected pre-canonicalization.
- Symlinks followed then re-validated against workspace root.

### Tool ACL Enforcement

- Analysis workers: `mutation_tools` not in their `bind_tools` list. Physically cannot call.
- Execution workers: `pool_tools` not in their list. Cannot spawn sub-workers (prevents runaway recursion).
- Orchestrator: full set minus `write_scratchpad` (orchestrator does not produce worker findings - consumes them).

### Secret Handling

- No credentials in prompts.
- LLM never sees env vars.
- Generated code scanned for accidental secret inclusion (post-hoc, non-blocking warning).

### Prompt Injection Defense

- File reads return structured `FileReadResult` - LLM sees JSON, not raw content in system-prompt position.
- Scratchpads validated against Pydantic schema - injected content cannot alter structure.
- Worker findings untrusted by orchestrator until validated.

### HITL as Last Line

Per-mutation HITL means even if agent reasoning is compromised, no filesystem mutation without human sign-off in `strict`/`standard` modes.

---

## 20. Open Questions

1. **Worker-initiated sub-workers?** Current design: workers cannot spawn. Keeps pool bounded. Accept this tradeoff? Alternative: allow max-depth-1 sub-spawn with separate budget.
2. **Cross-run error registry?** Single global `error_registry.json` across runs, or per-run? Global gives compounding learning but requires schema evolution.
3. **Plan re-synthesis limit?** If human rejects plan N times, what is the cap? Propose: 3 revisions, then escalate Tier 3.
4. **Postgres vs. JSON for scratchpads?** Currently JSON on disk. Large findings may benefit from Postgres JSONB. Deferred.
5. **Worker-side HITL?** Does an execution worker mutation also gate on HITL, or only orchestrator mutations? Proposal: yes, both. Worker mutation tools also behind `interrupt_before` (via subgraph compilation).
6. **Streaming worker findings?** Currently workers write findings atomically at end. Streaming partial findings (append mode) would improve orchestrator responsiveness - requires schema support for in-progress state.
7. **Dependency invalidation?** If a Phase 1 artifact changes mid-run, do analysis findings invalidate? Propose: hash-based detection at orchestrator iteration start; flag stale findings.
8. **Pool elasticity?** Fixed 5. Should it burst to 10 for small artifacts? Defer - start fixed.

---

## Appendix A - Phase State Machine

```
[setup] --setup_done--> [analysis]
                           |
                           | (worker + self-execution loop)
                           v
                      [synthesis] --plan_written--> [plan_review]
                                                      |
                                 (approved)<----------+---------->(rejected)
                                      |                               |
                                      v                               v
                                 [execution] <-------re-synth----- [analysis]
                                      |
                                      | (DAG wave dispatch loop)
                                      v
                                 [finalize] --> END
```

## Appendix B - Tool Description Style Guide

Every tool description MUST include:

- One-line purpose.
- Arg semantics (key fields).
- Failure modes (enumerate error codes).
- Idempotency guarantee (is retrying safe?).
- Concurrency note (does this acquire locks?).
- Example usage (1 short example).

Example:

```
edit_file

  Purpose: Replace a target string in a file on disk.
  Args:
    path: absolute path within workspace
    target: exact string to replace (replaces ONLY FIRST occurrence)
    replacement: new string
    expected_sha256: optional hash for optimistic concurrency
  Failure modes:
    - FILE_NOT_FOUND
    - TARGET_NOT_FOUND (returns suggestions via fuzzy match)
    - HASH_MISMATCH (file changed since last read)
    - PATH_TRAVERSAL
  Idempotent: no (string replacement is not idempotent).
  Concurrency: acquires file-level lock for write duration.
  Example:
    edit_file(path="src/foo.py", target="def old():", replacement="def new():")
```

## Appendix C - Worker Prompt Template Skeleton

```
System:
  You are a {worker_type} worker. Your job is: {job_description}.
  Assigned artifacts: {assigned_artifacts}.
  Scratchpad path: {scratchpad_path}.
  Tools available: {tool_names}.
  Context budget: {context_budget_tokens} tokens.
  When done, call write_scratchpad with structured findings matching the
  {FindingsSchema} schema, then release_artifact for each claimed artifact.
  You cannot spawn sub-workers. You cannot access other workers scratchpads.
  You cannot write to orchestrator memory files.
  Stop calling tools once findings are written and artifacts released.
```

## Appendix D - Orchestrator Prompt Skeleton

```
System:
  You are the lead autonomous developer. You orchestrate a team of worker agents
  and also perform work directly when appropriate.

  Current phase: {phase}
  Workspace: {workspace_path}
  Pool capacity: {MAX_CONCURRENT_WORKERS}
  Queue depth: {queue_depth}
  Free workers: {free_slots}

  Progress summary:
  {progress_summary}

  Registry status:
  Free artifacts: {free_artifacts_list}
  Claimed: {claimed_artifacts_list}

  Decision rules:
  - Spawn a worker when task is parallelizable and exceeds 2000 tokens of analysis.
  - Do it yourself when task is a single-file edit, a memory update, a registry query,
    or cross-cutting synthesis.
  - Queue do not block: spawn workers non-blocking, continue own work, await later.
  - Always release claimed artifacts.
  - Always update progress_summary after meaningful state changes.
  - Request human review ONLY for master plan approval or Tier-3 errors.
  - Do NOT attempt mutation tools unless approved task from master plan.

  Available tools: {tool_names_with_descriptions}
```

## Appendix E - Directory Layout Final

```
task_{id}/
  workspace/
    requirements/
      raw_prd.md
      enriched_prd.md
    hld/
      hld_*.md
    lld/
      lld_*.md
    user_stories/
      story_*.md
    generated_code/
      (project source)
  .nexus/
    registry/
      artifacts.json
      artifacts.json.lock
      worker_jobs.json
      worker_jobs.json.lock
    scratch/
      analysis/
        worker_{job_id}.json
      execution/
        worker_{job_id}.json
    memory/
      progress_full.md
      progress_summary.md
      session_log.jsonl
      master_plan.json
      final_report.md
      error_registry.json
    decisions/
      decision_{timestamp}_{id}.md
    file_locks/
      (file-level lock entries during mutation)
```

---

## Appendix F - Minimum Viable Implementation Order

Unchanged from team plan v1.0 Layer 0-4 approach, with adjustments:

- L0: Contracts (Pydantic models) - all models from Section 8.
- L1: Core utilities - `FileContextManager`, `FileLockManager`, `CodeMutatorPipeline`,
       `ArtifactRegistry`, `ScratchpadStore`, `AgentMemory`.
- L2: Tool bundles - typed tool groups with Pydantic args + Pydantic results.
- L3: ReAct engines - `OrchestratorAgent`, `WorkerAgent`.
- L4: LangGraph graph + Worker pool.
- L5: FastAPI surface + SSE streaming + Postgres wiring.

Build bottom-up. Each layer independently testable.

---

*End of architecture document. Next step: review, refine open questions, break into sprint tasks.*
