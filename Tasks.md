# Nexus Orchestrator — Consolidated Task Plan

**Parent Spec:** `nexus-orchestrator-lld.md` (v1.0)
**Mode:** Rapid delivery, agile, no mock-LLM tests
**Team:** 4 developers
**Branch:** `feat/nexus-orchestrator`

---

## 1. Goal

Deliver full LLD scope as one ordered backlog. Tasks executed strictly in dependency order. Each task has owner, complexity, acceptance criteria. No calendar, no sprint splits. Move fast, ship in dependency waves.

---

## 2. Complexity Scale

| Level | Marker | Meaning | Rough effort |
|---|---|---|---|
| Low | **L** | Simple, mechanical, well-understood | ≤ half day |
| Medium | **M** | Requires design thought + tests | 1-2 days |
| High | **H** | Multiple moving parts, integration risk | 3-5 days |
| Very High | **XH** | Cross-cutting, requires full focus | 1+ week |

---

## 3. Team Roles

| Dev | Focus |
|---|---|
| Dev 1 | Foundation Lead — contracts, navigation, validators, dependency mgmt |
| Dev 2 | Safety Engineer — file mutation, locks, terminal security |
| Dev 3 | Memory & Workspace — registry, scratchpads, memory, materializer |
| Dev 4 | Engine & Integration — ReAct, pool, LangGraph, API |

---

## 4. Definition of Ready (DoR)

- [ ] Acceptance criteria written
- [ ] Pydantic schema defined (if applicable)
- [ ] Dependencies listed
- [ ] Test plan listed
- [ ] Owner assigned

## 5. Definition of Done (DoD)

- [ ] Code merged to feature branch
- [ ] Unit tests passing (≥85% line coverage on new code)
- [ ] Integration tests where applicable (real components, not mocked LLM)
- [ ] Tool descriptions match LLD Appendix B style
- [ ] Pydantic round-trip serialization test green
- [ ] Type hints + docstrings on public API
- [ ] No new `# TODO` without GitHub issue
- [ ] Reviewed by ≥1 other dev
- [ ] Path traversal defense in any path-taking tool
- [ ] No `cat`/`echo`/`ls` in subprocess calls
- [ ] Logged via `setup_logging`

---

## 6. Ordering Principles

1. L0 contracts unlock everything → ship first.
2. L1 utilities unlock tool layer → ship second wave parallel.
3. Tool layer unlocks engines → third wave.
4. Engines + pool unlock graph → fourth wave.
5. Graph unlocks API + E2E → fifth wave.
6. Hardening + edge cases run in parallel after wave 4.

Within a wave, tasks parallelize across devs.

---

## 7. Master Task List (Dependency-Ordered)

### Phase A — Foundations (L0)

#### TASK-A1 — Pydantic models scaffold | **L** | Dev 1
Create `core/__init__.py`, `core/models.py`, `core/constants.py`, `core/errors.py`. Set up `model_config = ConfigDict(extra="forbid")` base.
- **Acceptance:** package imports, base model class compiles, enum literals defined per LLD §4.1.
- **Depends on:** none.

#### TASK-A2 — Registry models | **L** | Dev 1
`ArtifactRegistryEntry`, `ArtifactRegistry` with status validator + `model_validator` for lock consistency + done requires findings_path.
- **Acceptance:** invalid lock state raises ValueError; round-trip JSON preserves data.
- **Depends on:** TASK-A1.

#### TASK-A3 — Worker job models | **M** | Dev 1
`WorkerJobSpec` (discriminated by worker_type), `WorkerJobAck`, `WorkerJobStatus`, `PoolSnapshot`, `AwaitResult`, `PoolPersistedState`. Depth validator (0-2). Type-specific field requirement validators.
- **Acceptance:** depth=3 rejected; analysis spec without artifacts rejected; execution spec without task_id rejected.
- **Depends on:** TASK-A1.

#### TASK-A4 — Plan models with cycle detection | **M** | Dev 1
`TaskNode`, `DependencyEdge`, `MasterPlan`. Implement `_has_cycle` using topological sort. Validator catches dangling deps + cycles.
- **Acceptance:** 3-node cycle rejected; dangling `depends_on` rejected; complexity range 1-5 enforced.
- **Depends on:** TASK-A1.

#### TASK-A5 — HITL + tool result + memory models + error hierarchy | **M** | Dev 1
`HumanReviewRequest`, `ReviewDecision`, `MutationApproval`, `ToolResult` base + 14 subclasses, `ProgressEntry`, `DecisionLogEntry`, `ErrorRegistryEntry`, `ErrorEntry`, exception hierarchy in `core/errors.py` (Tier1/2/3, PathTraversal, DepthExceeded, etc.).
- **Acceptance:** every ToolResult subclass JSON-serializes; every exception correctly tiered.
- **Depends on:** TASK-A1.

---

### Phase B — Core Utilities (L1) — Parallel after Phase A

#### TASK-B1 — PathValidator + TokenBudget | **L** | Dev 1
Path canonicalization, traversal defense, terminal whitelist enforcement. `tiktoken` wrappers for count + trim_to.
- **Acceptance:** 10 traversal attack patterns from LLD §22.14 rejected; `validate_command("rm")` raises.
- **Depends on:** TASK-A1, TASK-A5.

#### TASK-B2 — FileContextManager | **M** | Dev 1
`read_chunk` with line-number injection (`f"{n:>6}: {line}"`), tiktoken count, SHA-256, pagination, FileNotFoundError suggests `list_directory`.
- **Acceptance:** 10K-line file with limit=200 returns 200 lines + truncated=true + next_offset=201; hash deterministic.
- **Depends on:** TASK-A1, TASK-B1.

#### TASK-B3 — FileLockManager | **M** | Dev 2
Per-file async locks via `O_CREAT | O_EXCL`, exp backoff (`min(0.5*2^n, 5.0)`, max wait 30s), TTL 60s with 30s grace, asynccontextmanager helper, holder validation on release.
- **Acceptance:** 10 concurrent acquires on same path → exactly 1 holds; stale lock auto-stolen; wrong-holder release raises.
- **Depends on:** TASK-A1.

#### TASK-B4 — CodeMutatorPipeline 6-Gate | **H** | Dev 2
Gate 1 markdown sanitizer (skip for `.md`), Gate 2 fuzzy match (exact → ws-normalized → stripped, first match only, 3 nearest suggestions), Gate 3 SHA-256 hash check, Gate 4 `.tmp` write via aiofiles, Gate 5 formatter via subprocess with `cwd=workspace_root` (non-fatal failure), Gate 6 atomic `os.replace`.
- **Acceptance:** each gate failure mode tested; `.tmp` always cleaned in failure path; FORMATTER_CRASH non-fatal.
- **Depends on:** TASK-A1, TASK-A5, TASK-B1, TASK-B3.

#### TASK-B5 — ArtifactRegistry | **M** | Dev 3
`init`, `claim`, `release`, `list_free`, `list_claimed_by`, `list_all`, `prune_expired`. All read-modify-write inside `FileLock(timeout=10)`. Schema versioning + migration hook. Stale claim auto-expiry. RegistryCorruptError on bad JSON.
- **Acceptance:** 10 concurrent claims → 1 winner; stale claim past TTL freed; corrupt JSON raises.
- **Depends on:** TASK-A1, TASK-A2.

#### TASK-B6 — ScratchpadStore | **L** | Dev 3
`create` (with header template per LLD §5.5), `append`, `write_header`, `read`, `list_for_phase`, `set_completion_summary`. Single-writer per file (job_id unique).
- **Acceptance:** header includes job_id, depth, artifacts, started_at; appends preserve content order.
- **Depends on:** TASK-A1, TASK-A3.

#### TASK-B7 — AgentMemory | **M** | Dev 3
`progress_full.md` append-only, `progress_summary.md` token-capped (rejects >600), `session_log.jsonl` one-object-per-line, `decisions/{id}.md` markdown writeup, `error_registry.json` upsert with `filelock` (cross-run shared at `NEXUS_GLOBAL_ERROR_REGISTRY_PATH`). All writes per-file `asyncio.Lock`. Schema migration on load.
- **Acceptance:** summary >600 tokens rejected with hint; concurrent writes from 3 processes don't corrupt; v0→v1 migration automatic.
- **Depends on:** TASK-A1, TASK-A5, TASK-B1.

#### TASK-B8 — PoolStatusStore | **L** | Dev 3
`record_spawn`, `record_status`, `mark_stale_running`, `snapshot`, `list_requeue_candidates`. Filelock on `worker_jobs.json`.
- **Acceptance:** crash + reload → RUNNING flips to STALE; concurrent updates serialize.
- **Depends on:** TASK-A1, TASK-A3.

---

### Phase C — Tool Layer (L2) — After Phase B

#### TASK-C1 — Tool registry + ACL resolver | **L** | Dev 1
`tool_registry.resolve(role, depth)` returns correct tool subset per LLD §6.1 ACL matrix. Module exports `TOOLS` + `TOOL_NAMES` per group.
- **Acceptance:** analysis worker excludes mutation_tools; depth=2 worker excludes pool_tools; orchestrator gets full surface minus write_scratchpad.
- **Depends on:** TASK-A1.

#### TASK-C2 — Navigation tools | **M** | Dev 1
`list_directory` (filters `.git`, `node_modules`, `__pycache__`, `.venv`, `dist`, `build`, `.nexus`), `glob_search` (rglob), `grep_search` (Python `re`, no rg dep), `read_file_chunk` (delegates FileContextManager). All `StructuredTool.from_function`. Path traversal defense in every tool.
- **Acceptance:** filters work; grep truncates at max_results with truncated=true; all results return correct ToolResult subclass.
- **Depends on:** TASK-A1, TASK-A5, TASK-B1, TASK-B2, TASK-C1.

#### TASK-C3 — Mutation tools | **M** | Dev 2
`create_file`, `edit_file`, `run_terminal`. All routed to `tool_node_mutating`. Terminal whitelist + per-arg path validation + 2K char output truncation + timeout via `asyncio.create_subprocess_exec` kill on TimeoutError. `edit_file` description explicit "Replaces ONLY FIRST occurrence".
- **Acceptance:** `run_terminal("cat", ...)` returns TERMINAL_WHITELIST_VIOLATION; timeout kills subprocess; create+edit delegate to CodeMutatorPipeline.
- **Depends on:** TASK-A1, TASK-A5, TASK-B1, TASK-B3, TASK-B4, TASK-C1.

#### TASK-C4 — Scratchpad tools | **L** | Dev 3
`write_scratchpad` (worker only, append mode), `read_scratchpad` (shared), `list_scratchpads`, `complete_job` (writes `## Completion Summary`, sets exit_code, signals worker exit).
- **Acceptance:** complete_job sets finished_at; worker without complete_job marked FAILED with `did_not_complete`.
- **Depends on:** TASK-A1, TASK-B6, TASK-C1.

#### TASK-C5 — Memory tools | **L** | Dev 3
`append_progress_full`, `write_progress_summary` (token-cap enforced), `log_decision`, `lookup_known_error`. Orchestrator-only ACL on first two; shared on log_decision + lookup.
- **Acceptance:** summary cap rejects oversize content; lookup returns None on miss + entry on hit.
- **Depends on:** TASK-A1, TASK-A5, TASK-B7, TASK-C1.

#### TASK-C6 — Registry tools | **L** | Dev 3
`claim_artifact`, `release_artifact`, `list_free_artifacts`, `list_claimed_artifacts`. Holder = ctx.holder injected by tool node.
- **Acceptance:** claim of locked artifact returns `already_locked_by`; release with wrong holder raises.
- **Depends on:** TASK-A1, TASK-A5, TASK-B5, TASK-C1.

#### TASK-C7 — Pool tools | **M** | Dev 4
`spawn_worker` (depth check, queue saturation check, generates UUID4 job_id, enqueues, returns ack), `check_worker_status`, `await_workers` (blocks on completion events with timeout), `list_workers`, `cancel_worker` (sets cancellation event).
- **Acceptance:** depth=2 spawn returns DEPTH_EXCEEDED; queue ≥50 returns POOL_SATURATED; await returns AwaitResult with timed_out flag.
- **Depends on:** TASK-A1, TASK-A3, TASK-A5, TASK-D2 (WorkerPool); can stub initially.

#### TASK-C8 — HITL tools | **L** | Dev 4
`request_human_review` raises `GraphInterrupt` with HumanReviewRequest payload. `write_master_plan` validates DAG, persists to `.nexus/memory/master_plan.json`, increments plan_revision, raises Tier-3 if revision > 3.
- **Acceptance:** request_human_review surfaces correct payload via interrupt; write_master_plan rejects cyclic plan; revision >3 → PlanRevisionExhaustedError.
- **Depends on:** TASK-A1, TASK-A4, TASK-A5, TASK-C1.

#### TASK-C9 — Tool description audit | **L** | Dev 1
Every tool description has all 6 sections (Purpose, Args, Failure modes, Idempotent, Concurrency, Example). Test `test_tool_descriptions_completeness.py`.
- **Acceptance:** test fails if any tool missing a section.
- **Depends on:** TASK-C2..C8.

---

### Phase D — Engines + Pool (L3)

#### TASK-D1 — OrchestratorAgent | **H** | Dev 4
Class with `__init__(llm_provider, memory, registry, pool, tools, prompt_template, max_iterations_per_phase, max_total_iterations)`. `run(state)` = one ReAct cycle (1 LLM call only, no internal loop). System prompt template injection (progress_summary + registry snapshot + pool snapshot). `@llm_retry_decorator` for Tier-1. Phase iteration cap → Tier-3 error + route to error phase.
- **Acceptance:** 1 invocation = 1 LLM call; iteration cap appends Tier-3 error; tool calls emit stream events.
- **Depends on:** TASK-A1, TASK-A5, TASK-B7, TASK-B5, TASK-C1..C8.

#### TASK-D2 — WorkerAgent + worker subgraph builder | **H** | Dev 4
`WorkerAgent` class with phase-specific tool ACL via tool_registry. `build_worker_graph` factory builds StateGraph + ToolNode (split safe/mutating per worker_type). Worker subgraph compiled with `interrupt_before=["tool_node_mutating"]` for execution workers, own thread_id. Scratchpad header write on init. Iteration cap + timeout enforcement. complete_job triggers natural exit. Cleanup on cancel/timeout/error releases artifacts (always-runs `finally`).
- **Acceptance:** worker reads artifact, writes scratchpad, calls complete_job, releases claim; cancellation propagates within 1 iteration; all claims released even on exception.
- **Depends on:** TASK-A1, TASK-A3, TASK-A5, TASK-B5, TASK-B6, TASK-C1..C8.

#### TASK-D3 — WorkerPool | **XH** | Dev 4
`WorkerPool(capacity=5, max_queue_depth=50, worker_factory, status_store, registry, stream_emitter, global_pool_timeout_s)`. `start` spawns 5 slot coroutines. `_slot_runner` with `asyncio.timeout(spec.timeout_s)`, always-release `finally`, status persisted to `PoolStatusStore` per transition. `enqueue` validates depth + queue depth. `cancel`, `snapshot`, `get_completion_event`, `rehydrate` (marks STALE on resume — no auto-requeue, orchestrator decides).
- **Acceptance:** capacity-5 + 6 spawn → 5 running, 1 queued; timeout → status timed_out + claim released; crash + restart → RUNNING flipped to STALE; sub-worker depth enforced; queue overflow returns POOL_SATURATED.
- **Depends on:** TASK-A3, TASK-A5, TASK-B5, TASK-B8, TASK-D2.

#### TASK-D4 — WorkspaceMaterializer | **M** | Dev 3
Validates `hld_details`/`lld_details` not None (fail fast). Materializes inputs to `workspace/` (requirements/, hld/, lld/, user_stories/). Initializes `.nexus/` skeleton. Builds initial registry via SHA-256 per artifact. Re-run handling: backup existing as `task_{id}_prev_{timestamp}`.
- **Acceptance:** None hld_details raises Tier-3; second run backs up first; SHA-256 stable.
- **Depends on:** TASK-A1, TASK-A2, TASK-B5.

#### TASK-D5 — setup_workspace_node + finalize_node | **L** | Dev 3
`setup_workspace_node` invokes WorkspaceMaterializer + emits workspace_ready event. `finalize_node` drains pool gracefully, releases all still-claimed artifacts, writes `final_report.md` (per-phase iteration counts + completion status + files changed), emits run_complete.
- **Acceptance:** setup creates correct dir tree; finalize releases orphaned claims; final report includes all phase summaries.
- **Depends on:** TASK-D3, TASK-D4.

---

### Phase E — Graph + API (L4)

#### TASK-E1 — DevPhaseState + reducers + constants | **M** | Dev 4
`DevPhaseState` TypedDict per LLD §10. `add_messages_trimmed` reducer wrapping `add_messages` + `trim_messages(max_tokens=MESSAGE_WINDOW_TOKENS, strategy="last", start_on="human", include_system=True)`. `dev_phase_constants.py` with all env-backed limits per LLD §16.1. `automation_agent_role_enum.py` adds `AUTONOMOUS_DEV`.
- **Acceptance:** state TypedDict validates; reducer trims oldest first; all limits configurable via env.
- **Depends on:** TASK-A1.

#### TASK-E2 — Settings + Postgres lifespan | **M** | Dev 4
`config/settings.py` Pydantic BaseSettings loading all env vars per LLD §19. `api/main.py` lifespan opens `AsyncConnectionPool` + `AsyncPostgresSaver`, calls `await saver.setup()` to auto-create tables. Graceful shutdown closes pool.
- **Acceptance:** lifespan starts/stops cleanly; tables auto-created; pool size respects POSTGRES_POOL_MIN/MAX.
- **Depends on:** TASK-A1.

#### TASK-E3 — Graph builder + conditional router + interrupts | **H** | Dev 4
`build_dev_phase_graph` adds 6 nodes (setup_workspace, orchestrator_agent, tool_node_safe, tool_node_mutating, tool_node_hitl, finalize). Conditional router `route_from_orchestrator`: empty tool_calls → finalize; mutation_tools intersect → mutating; hitl_tools intersect → hitl; else → safe. Compile with `interrupt_before=["tool_node_mutating", "tool_node_hitl"]` per HITL_MODE config (strict/standard/relaxed).
- **Acceptance:** graph compiles; router correctly dispatches; interrupt fires on mutation; resume via `Command(resume=...)` advances graph; HITL_MODE=relaxed disables Gate 2.
- **Depends on:** TASK-D1, TASK-D2, TASK-D3, TASK-D5, TASK-E1, TASK-E2.

#### TASK-E4 — FastAPI endpoints | **M** | Dev 4
`POST /dev-run` (start thread_id, returns initial state or pending_interrupt), `POST /dev-run/{thread_id}/resume` (accepts ReviewDecision or MutationApproval), `GET /dev-run/{thread_id}/state` (sanitized snapshot — no full plan, no scratchpad contents), `POST /dev-run/{thread_id}/cancel`, `GET /dev-run/{thread_id}/final-report`. All requests/responses Pydantic with `extra="forbid"`.
- **Acceptance:** invalid resume payload → 422; cancel sets cancellation event on running pool; state endpoint excludes large fields.
- **Depends on:** TASK-E3.

#### TASK-E5 — SSE streaming endpoint + StreamEmitter | **M** | Dev 4
`GET /dev-run/{thread_id}/stream` via `sse-starlette`. `StreamEmitter` wraps `stream_writer.create_emitter()`. All 13 event types per LLD §17.1 emitted from orchestrator + pool + tool node + memory layer.
- **Acceptance:** stream emits ≥5 event types during a sample run; terminates on phase=finalize or error.
- **Depends on:** TASK-E4.

#### TASK-E6 — Worker-side HITL wiring | **M** | Dev 2 + Dev 4
Worker subgraph compiled with own thread_id and `interrupt_before=["tool_node_mutating"]` for execution workers. Pool slot detects interrupt, sets WorkerJobStatus.status="interrupted", emits `worker_mutation_request` event. New API endpoint `POST /dev-run/{thread_id}/workers/{job_id}/resume` accepts MutationApproval → resumes worker subgraph. `WorkerStatus` enum extended with "interrupted".
- **Acceptance:** worker mutation triggers interrupt; UI receives event with worker_id + pending tool_calls; resume continues worker.
- **Depends on:** TASK-D2, TASK-D3, TASK-E3.

---

### Phase F — Hardening + Observability — Parallel after Phase E

#### TASK-F1 — Plan revision rejection loop | **L** | Dev 4
Synthesizer reads ReviewDecision on rejection, applies edits, increments plan_revision. write_master_plan checks `plan_revision > PLAN_MAX_REVISIONS` → Tier-3 PlanRevisionExhaustedError → HITL Gate 3 escalation.
- **Acceptance:** 4th rejected revision → Tier-3 with escalation payload.
- **Depends on:** TASK-C8, TASK-E3.

#### TASK-F2 — Error tier handler + poison pill | **M** | Dev 4
`@llm_retry_decorator` tenacity wrapping for Tier-1. Tier-2 errors wrapped in ToolResult(success=false). Per-fingerprint counter on agent + worker; ≥3 same-fingerprint failures → poison status → orchestrator notified. Tier-3 → state.errors append + HITL Gate 3 or terminate.
- **Acceptance:** Tier-1 retry exhaust → Tier-2 escalation; 3-strike same fingerprint → poison; security errors always Tier-3.
- **Depends on:** TASK-A5, TASK-D1, TASK-D2.

#### TASK-F3 — Stream events + agent_scope metadata | **L** | Dev 4
All nodes get `agent_scope(...)` via `workflow.add_node(... , metadata=agent_scope(...))`. All tool nodes + pool emit events per LLD §17.1.
- **Acceptance:** every event tagged with agent_scope; LangSmith traces show correct hierarchy.
- **Depends on:** TASK-E3, TASK-E5.

#### TASK-F4 — Prometheus metrics endpoint | **M** | Dev 4
`/metrics` endpoint exposing counters/gauges/histograms per LLD §17.2. Hook into stream events.
- **Acceptance:** `nexus_tool_latency_ms` histogram populated; `nexus_pool_queue_depth` gauge live.
- **Depends on:** TASK-E5.

#### TASK-F5 — Secret scanning post-mutation hook | **L** | Dev 2
Regex scan post-mutation for AWS keys, Google API keys, JWT, private keys. Match → Tier-2 event (non-blocking).
- **Acceptance:** test artifact with planted secret triggers warning; mutation still completes.
- **Depends on:** TASK-C3.

#### TASK-F6 — Health + ready endpoints | **L** | Dev 4
`GET /healthz` (checkpointer + pool + registry writable), `GET /readyz` (graph compiled).
- **Acceptance:** health 503 on Postgres down; ready 200 after lifespan complete.
- **Depends on:** TASK-E2, TASK-E4.

---

### Phase G — Multi-Worker Migration Prep

#### TASK-G1 — Postgres artifact registry backend | **H** | Dev 3
Implement `PostgresArtifactRegistry` satisfying same interface as JSON registry. Schema per LLD §13.3. Claim via `SELECT FOR UPDATE` with TTL expiry check in WHERE clause. Toggle via `REGISTRY_BACKEND=postgres` env.
- **Acceptance:** swap JSON → postgres without orchestrator/worker code change; concurrent claim across processes → 1 winner.
- **Depends on:** TASK-B5, TASK-E2.

#### TASK-G2 — Sticky routing config + uvicorn multi-worker validation | **M** | Dev 4
Document Nginx/Traefik config with `hash $arg_thread_id consistent`. Validate `uvicorn --workers 4` works with thread_id routing. Pool rehydration on worker startup scans worker_jobs.json for owned threads.
- **Acceptance:** 4-worker uvicorn with sticky routing handles 10 concurrent runs without state corruption.
- **Depends on:** TASK-D3, TASK-E2, TASK-G1.

---

### Phase H — Real-LLM E2E + Edge Cases

#### TASK-H1 — Real-LLM E2E: full analysis flow | **H** | Dev 4
Run real LLM through `setup → analysis → synthesis → plan_review (interrupt)`. Verify: 3 workers spawn, queue overflow at 6th, all workers complete, scratchpads aggregated, plan written.
- **Acceptance:** real PRD + 2 HLDs + 3 LLDs analyzed end-to-end; plan validated; interrupt fires.
- **Depends on:** all Phase E.

#### TASK-H2 — Real-LLM E2E: plan-review HITL approve + resume | **M** | Dev 4
Plan review interrupt. Approve via API. Graph advances to execution phase.
- **Acceptance:** approval transitions phase; rejection cycles back; 3-revision cap fires correctly.
- **Depends on:** TASK-H1, TASK-F1.

#### TASK-H3 — Real-LLM E2E: execution phase with mutation HITL | **H** | Dev 4
Approved plan → execution dispatch → first wave workers → mutation tool call → interrupt → human approves → mutation lands → file on disk.
- **Acceptance:** at least 1 file mutated end-to-end with HITL gate; secrets scan emits warning if planted; rejected mutation → synthetic ToolMessage.
- **Depends on:** TASK-H2, TASK-E6, TASK-F5.

#### TASK-H4 — Resume after crash E2E | **M** | Dev 4
Kill uvicorn mid-run → restart → resume same thread_id → state preserved → flow completes.
- **Acceptance:** kill at every phase boundary recoverable; pool rehydrates; STALE workers visible to orchestrator.
- **Depends on:** TASK-H1, TASK-D3.

#### TASK-H5 — Concurrency stress test | **M** | Dev 2 + Dev 3
10 concurrent registry claims (1 winner); 10 concurrent file locks (1 holder); 5 worker pool with 20 spawn calls (5 running, 15 queued); pool saturation at queue=50.
- **Acceptance:** no corruption under load; no orphan claims; backpressure correct.
- **Depends on:** TASK-B3, TASK-B5, TASK-D3.

#### TASK-H6 — Edge cases catalog (LLD §22.1-22.30) | **H** | All devs
One ticket per edge case. Owner per LLD owner mapping. Test or defensive code added for each. Defer truly impossible-to-trigger to Phase I.
- **Acceptance:** 30 edge cases each have a reproduction test or explicit "WONTFIX" rationale.
- **Depends on:** all Phase E.

---

### Phase I — Ops + Polish

#### TASK-I1 — Prompt engineering | **H** | Dev 4 + Tech Lead
Final orchestrator prompt + analysis worker prompt + execution worker prompt. Iterate against real-LLM E2E results until convergence on representative tasks.
- **Acceptance:** orchestrator chooses spawn vs self correctly ≥80% on test corpus; worker scratchpads contain actionable findings; plan synthesis produces valid DAGs.
- **Depends on:** TASK-H1.

#### TASK-I2 — Error registry schema migration scripts | **L** | Dev 3
`scripts/migrate_error_registry.py`. Migration v0→v1 stub. Backup-before-migrate via `error_registry.json.bak.v{from}`.
- **Acceptance:** schema bump migration is reversible (backup retained).
- **Depends on:** TASK-B7.

#### TASK-I3 — Deployment runbook + Docker images | **M** | Dev 4
Dockerfile, docker-compose with Postgres, env example, single-worker + multi-worker run instructions, graceful shutdown verification.
- **Acceptance:** `docker compose up` boots full stack; `docker compose down` graceful.
- **Depends on:** TASK-E2, TASK-F6.

#### TASK-I4 — LangSmith tracing verification | **L** | Dev 4
Confirm parent-child run linkage between orchestrator and worker subgraphs in LangSmith UI.
- **Acceptance:** worker traces show `parent_run_id` linkage; sub-worker shows depth-2 nesting.
- **Depends on:** TASK-D2, TASK-F3.

#### TASK-I5 — Final security audit | **M** | Dev 2
Path traversal fuzzing across every tool. Whitelist bypass attempts. Prompt injection test corpus. ACL bypass attempts (worker calling forbidden tool).
- **Acceptance:** zero new findings of severity ≥ Medium.
- **Depends on:** all Phase F.

#### TASK-I6 — Documentation pass | **L** | All
README, contribution guide, inline docstrings audit, env var docs.
- **Acceptance:** new dev can boot stack and run sample task in ≤30 minutes from README.
- **Depends on:** TASK-I3.

---

## 8. Dependency Summary

```
A1 ──► A2, A3, A4, A5
A1 + B1 ──► B2
A1 ──► B3
A1 + A5 + B1 + B3 ──► B4
A1 + A2 ──► B5
A1 + A3 ──► B6
A1 + A5 + B1 ──► B7
A1 + A3 ──► B8

A1 ──► C1
A1+A5+B1+B2+C1 ──► C2
A1+A5+B1+B3+B4+C1 ──► C3
A1+B6+C1 ──► C4
A1+A5+B7+C1 ──► C5
A1+A5+B5+C1 ──► C6
A1+A3+A5+(D3 stub)+C1 ──► C7
A1+A4+A5+C1 ──► C8
C2..C8 ──► C9

A1+A5+B5+B7+C1..C8 ──► D1
A1+A3+A5+B5+B6+C1..C8 ──► D2
A3+A5+B5+B8+D2 ──► D3
A1+A2+B5 ──► D4
D3+D4 ──► D5

A1 ──► E1
A1 ──► E2
D1+D2+D3+D5+E1+E2 ──► E3
E3 ──► E4
E4 ──► E5
D2+D3+E3 ──► E6

(parallel after E)
C8+E3 ──► F1
A5+D1+D2 ──► F2
E3+E5 ──► F3
E5 ──► F4
C3 ──► F5
E2+E4 ──► F6

B5+E2 ──► G1
D3+E2+G1 ──► G2

(after Phase E)
all_E ──► H1
H1+F1 ──► H2
H2+E6+F5 ──► H3
H1+D3 ──► H4
B3+B5+D3 ──► H5
all_E ──► H6

H1 ──► I1
B7 ──► I2
E2+F6 ──► I3
D2+F3 ──► I4
all_F ──► I5
I3 ──► I6
```

---

## 9. Parallelization Map (per phase)

- **Phase A:** Dev 1 owns all (sequential A1 → A2/A3/A4/A5 parallel after A1).
- **Phase B:** All 4 devs in parallel — Dev 1 (B1, B2), Dev 2 (B3, B4), Dev 3 (B5, B6, B7, B8), Dev 4 starts engine scaffolding + studies LangGraph patterns.
- **Phase C:** All 4 devs parallel — Dev 1 (C1, C2, C9), Dev 2 (C3), Dev 3 (C4, C5, C6), Dev 4 (C7, C8).
- **Phase D:** Dev 4 owns D1, D2, D3 (critical path); Dev 3 owns D4, D5 in parallel.
- **Phase E:** Dev 4 owns most; Dev 2 pairs on E6.
- **Phase F:** All devs parallel ownership per task.
- **Phase G:** Dev 3 (G1), Dev 4 (G2).
- **Phase H:** All devs contribute, Dev 4 leads.
- **Phase I:** Dev 4 leads I1, I3, I4, I5; Dev 3 (I2); all (I6).

---

## 10. Hard Sync Points

- After **TASK-A5:** all models merged. Dev 2 + Dev 3 unblocked.
- After **TASK-B7:** memory subsystem usable. Tool layer unblocked.
- After **TASK-C9:** tool layer audited + complete. Engine layer unblocked.
- After **TASK-D3:** worker pool stable. Graph builder unblocked.
- After **TASK-E5:** graph + API + streaming live. Real-LLM E2E unblocked.

---

## 11. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-1 | Models (A1-A5) slip past Phase A | MED | HIGH | Dev 1 priority; pair with Dev 4 if slipping |
| R-2 | Postgres checkpointer setup issues | MED | MED | Dev 4 spike on Day 1; surface early |
| R-3 | LangGraph version churn | LOW | MED | Pin `langgraph>=0.2.50,<0.3` |
| R-4 | Worker pool deadlock under stress | MED | HIGH | TASK-H5 stress test; cancellation hard-kill fallback |
| R-5 | Filelock unreliable on Windows | LOW | MED | `filelock` lib cross-platform; CI matrix Win + Linux |
| R-6 | Real-LLM cost during E2E | MED | LOW | Use cheapest model for Phase H; cache responses |
| R-7 | Sub-worker depth edge cases | LOW | MED | Property-based test in TASK-D3 |
| R-8 | HITL UI integration drift | MED | MED | Lock SSE event schema early in TASK-E5 |
| R-9 | Plan synthesis quality (DAG validity) | MED | HIGH | TASK-I1 prompt iteration with real corpus |
| R-10 | Person sick/PTO | MED | HIGH | Pair on critical path; cross-train Day 1 |

---

## 12. Branch + Merge Strategy

- Main feature branch: `feat/nexus-orchestrator`
- Per-task branch: `feat/nexus-{task-id}-{slug}`
- Merge order enforced by §8 dependency graph
- Every merge requires ≥1 reviewer + green CI
- Daily integration to feature branch after standup
- Final merge to `main` after TASK-I5 + I6 sign-off

---

## 13. Acceptance — Project Done

Project complete when:

- [ ] All Phase A-I tasks Done per DoD
- [ ] Real-LLM E2E (TASK-H1 + H2 + H3) passes
- [ ] Crash + resume (TASK-H4) passes at every phase boundary
- [ ] Concurrency stress (TASK-H5) clean
- [ ] All 30 edge cases (TASK-H6) addressed
- [ ] Multi-worker uvicorn (TASK-G2) verified
- [ ] Security audit (TASK-I5) zero ≥Medium findings
- [ ] Deployment runbook (TASK-I3) reproducible
- [ ] README onboarding ≤30 min (TASK-I6)

---

## 14. Daily Standup Template

```
Each dev:
1. Done since last standup: [task ids]
2. Doing now: [task id + sub-step]
3. Blockers: [name + person needed]
4. Risk flags: [any new risks not in §11]
```

---

## 15. Out-of-Scope (explicit)

- Real-time collaborative editing
- Distributed multi-node worker pool
- Custom LLM gateway (use existing LLMProvider)
- Generic tool plugin system
- Post-execution code review or test running (downstream phase)
- Git operations (existing dev_code_saver)
- Cross-project artifact sharing
- Streaming worker findings (resolved NO)
- Artifact change detection mid-run (resolved NO)
- Pool bursting (resolved NO, fixed at 5)
- Mock-LLM tests (per team direction)

---

*End of Task Plan. Kickoff: confirm Phase A start. Track via GitHub Issues, label per Phase.*
