# AGENTS.md - bd Control-Plane Bootloader

This file is the operating system for AI agent behavior in this repo.
It exists to enforce deterministic execution over probabilistic model behavior.

## Purpose

Use this workflow to prevent:
- intent drift
- scope creep
- context loss across sessions
- unverifiable completion claims
- coordination and dependency-state failures

Success means:
- deterministic state transitions in `bd`
- verification evidence recorded
- safe close reasons
- resumable handoff
- clean landing state

## Failure Patterns Mitigated

This file is designed to prevent these common agent failures:
- intent drift from user request
- unplanned scope expansion
- context loss across turns/sessions
- unverifiable "done" claims
- multi-agent overlap/race conditions
- task graph wiring errors (missing links/cycles/orphans)
- coding before intake mapping/audit gates
- unsafe closes without acceptance evidence
- dependency-state corruption from unsafe close reasons
- liveness stalls from deferred/blocked work not resurfacing
- interactive/destructive command misuse in automation
- weak handoff artifacts with no exact next command

## Authority Order

When instructions conflict, resolve in this order:
1. user instruction
2. `bd` source behavior and `bd <cmd> --help`
3. the inlined Control-Plane Contract section in this file
4. split-agent docs (`claude-plugin/agents/*.md`, `docs/agents/*.md`)
5. this file

When conflict handling is required, record:
- `Conflict: <sources>; Resolution: <decision>`

## Ownership Boundary

- `bd` deterministic owner:
  - preflight gates
  - lifecycle transitions
  - intake audits
  - recovery loop/signature
  - landing gates
  - reason lint and result envelopes
- split-agent judgment owner:
  - decomposition strategy
  - prioritization and tradeoffs
  - architecture decisions
  - handoff writing quality

## Control-Plane Contract (Inlined)

Deterministic `bd` command surfaces:
- `flow claim-next`
- `flow create-discovered`
- `flow block-with-context`
- `flow close-safe`
- `flow transition`
- `intake audit`
- `intake map-sync`
- `intake planning-exit`
- `intake bulk-guard`
- `preflight gate`
- `preflight runtime-parity`
- `recover loop`
- `recover signature`
- `resume`
- `land`
- `reason lint`

All control-plane commands in JSON mode are expected to emit:

```json
{
  "ok": true,
  "command": "flow claim-next",
  "result": "claimed",
  "issue_id": "bd-123",
  "details": {},
  "recovery_command": "bd ready --limit 5",
  "events": ["claimed"]
}
```

Envelope fields:
- `ok`: boolean command outcome
- `command`: canonical command identifier
- `result`: deterministic result-state enum
- `issue_id`: issue identifier when relevant
- `details`: structured context payload
- `recovery_command`: remediation command when applicable
- `events`: deterministic event tags

Common result values (non-exhaustive):
- `claimed`
- `wip_blocked`
- `no_ready`
- `contention`
- `policy_violation`
- `partial_state`
- `invalid_input`
- `system_error`
- `gate_failed`
- `operation_failed`
- `check_passed`
- `landed_with_skipped_gate3`
- `ok`

Exit codes:
- `0`: success or non-fatal deterministic state outcome
- `1`: generic command/system error
- `3`: policy violation
- `4`: partial state

## Cold Start (Run First)

Before any claim or lifecycle write:

```bash
bd where
bd preflight gate --action claim --json
bd ready --limit 5 --json
```

If preflight does not return pass, do not claim or write. Remediate first.

## State Machine

Required execution order:

`BOOT -> PLANNING -> INTAKE -> EXECUTING <-> RECOVERING -> LANDING -> END`

Abort path is always available:

`* -> ABORT -> END`

Do not reorder stages.

## Transition Rules

State transitions are triggered by specific conditions and must be recorded deterministically.

| Transition Type | Trigger Condition | Action | Next State |
|----------------|-------------------|--------|------------|
| claim_failed | Claim attempt returned error | Log error, try next candidate | EXECUTING |
| claim_became_blocked | Issue became blocked after claim | Record blocker, update status | RECOVERING |
| exec_blocked | Execution hit external blocker | Run block-with-context | RECOVERING |
| test_failed | Verification command failed | Record failure, determine remediation | RECOVERING |
| conditional_fallback_activate | Conditional dependency activated | Activate fallback task | EXECUTING |
| priority_poll | P0 work appeared | Determine preemption | EXECUTING |
| transient_failure | Temporary failure (network, resource) | Retry with backoff | EXECUTING |
| priority_preempt | Higher priority work appeared | Defer current, claim P0 | EXECUTING |
| session_abort | Unrecoverable condition | Write abort handoff, exit | END |
| decomposition_invalid | Child tasks don't satisfy parent | Rewire dependencies | PLANNING |

Trigger transitions with:

```bash
bd flow transition --type <type> --issue <id> --reason "<why>"
```

## Mandatory Write Policy

For lifecycle transitions, use `bd flow` wrappers:
- `bd flow claim-next`
- `bd flow create-discovered`
- `bd flow block-with-context`
- `bd flow close-safe`
- `bd flow transition`

Do not use interactive lifecycle mutation paths.
Do not use `bd edit`.

## Deterministic Execution Loop

1. Select candidate from `bd ready`.
2. Run preclaim lint:
   - `bd flow preclaim-lint --issue <id>`
3. Claim:
   - `bd flow claim-next ...`
4. Execute and verify required behavior.
5. Close with strict checks:
   - `bd flow close-safe --issue <id> --reason "<safe reason>" --verified "<command+result>" --require-traceability --require-spec-drift-proof --require-priority-poll --require-parent-cascade`

Never claim tests/commands ran unless they were executed.

## Discovery Quarantine Tiers

When discovering new work during execution, classify into tiers:

| Tier | Scope | Time | Action |
|------|-------|------|--------|
| T1 | Trivial, no new files | <5 min | Inline fix, no tracking |
| T2 | Small, 1-2 files | <15 min | Inline with note in issue |
| T3 | Medium, needs tracking | <60 min | Create task with discovered-from link |
| T4 | Large, multi-session | >60 min | Create epic, defer to planning |

**Boundary Rules:**
- If unsure between T2/T3: choose T3 (track it)
- T3+ requires `bd flow create-discovered --from <origin-id> --title "..."`
- T4 stops current execution, records state, escalates to planning

**Rationale:** Unconstrained discovery causes scope creep and context loss. Tier boundaries force explicit decisions about inline vs. tracked work.

## Intake Hard Gate

If plan items are 2+ or decomposition creates 4+ tasks:
- complete mapping/audit before claim or coding
- require `INTAKE_AUDIT=PASS`

Use:

```bash
bd intake map-sync ...
bd intake audit --epic <id> --write-proof --json
```

## Recover Loop (When Scoped Ready Is Empty)

Run deterministic recovery before declaring idle:

```bash
bd recover loop --parent <epic-id> --module-label module/<name> --json
bd recover signature --parent <epic-id> --iteration <n> --elapsed-minutes <m> --json
```

### Recovery Loop Phases

When `bd ready` returns empty, run `bd recover loop` which executes:

**Phase 1: Quick Diagnosis**
- Check preflight gate status
- Query scoped ready set
- Result: Either find work or proceed to Phase 2

**Phase 2: Structural Diagnosis**
- Check for dependency cycles: `bd dep cycles`
- Find stale WIP (in_progress >24h): `bd stale --days 1`
- Review dep trees for blocked chains: `bd dep tree <id> --direction up`
- Result: Identify structural blockers or proceed to Phase 3

**Phase 3: Limbo Detection**
- Find invisible blockers (unlinked dependencies)
- Check external dependencies (waiting on PR, review, response)
- Detect deferred work without resurface date
- Result: Identify limbo state or proceed to Phase 4

**Phase 4: Widen Scope**
- Remove parent/label filters
- Query full ready set
- Check other modules/epics for work
- Result: Find work or proceed to Phase 5

**Phase 5: Convergence/Escalation**
- Calculate convergence signature: iteration count + elapsed time
- If cycles < 3 AND elapsed < 30min: continue loop
- If cycles >= 3 OR elapsed >= 30min: escalate with decision request
- Result: Escalation with full context

**Convergence Signature**: `bd recover signature --parent <id> --iteration <n> --elapsed-minutes <m>`

Interpret results:
- `recover_ready_found` or `recover_ready_found_widened`: return to execute loop
- `recover_limbo_detected`: stay in recovery until resolved
- signature escalation required: escalate with explicit decision request

## Landing

Use deterministic landing engine:

```bash
bd land --epic <epic-id> --require-quality --quality-summary "<tests|lint|build>" --require-handoff --next-prompt "<prompt>" --stash "<none|restore>" --pull-rebase --sync --push --json
```

Result handling:
- `landed`: complete
- `landed_with_skipped_gate3`: partial; include explicit skip rationale in handoff

## Abort Runbook

For unrecoverable conditions:

1. Writable abort:
   - `bd flow transition --type session_abort --issue "<id-or-empty>" --reason "<why>" --context "<state summary>" --abort-handoff ABORT_HANDOFF.md`
2. No-write abort:
   - `bd flow transition --type session_abort --reason "<why>" --context "<state summary>" --abort-handoff ABORT_HANDOFF.md --abort-no-bd-write`
3. If `bd` cannot run:
   - write `ABORT_HANDOFF.md` manually with reason, state, touched files, exact recovery commands

## Close-Reason Safety

Success close reasons must be safe and deterministic.
Do not include failure-trigger keywords in success reasons.

Use:

```bash
bd reason lint --reason "<close reason>"
```

### Trigger Keywords List

The following 11 keywords are treated as failure triggers by `bd` and are used to evaluate conditional-blocks dependencies (where task B runs only if task A fails):

- `failed`
- `rejected`
- `wontfix`
- `won't fix`
- `canceled`
- `cancelled`
- `abandoned`
- `blocked`
- `error`
- `timeout`
- `aborted`

These keywords must NOT appear in success close reasons. Including them corrupts conditional-blocks dependency evaluation, causing false failure detection.

### Success Close Verbs

When closing tasks successfully, use these verbs (and similar action verbs) to describe completion:

- `added` - New feature, code, or configuration added
- `implemented` - Feature or behavior implemented
- `refactored` - Code restructured without behavior change
- `updated` - Existing code or configuration modified
- `removed` - Code or configuration deleted
- `migrated` - Data or code moved between systems
- `configured` - Settings or infrastructure configured
- `extracted` - Code pulled out into module/function
- `replaced` - Old implementation swapped for new
- `consolidated` - Multiple elements combined into one

### Safe vs Unsafe Examples

**SAFE (success close reasons):**

- `implemented: added auth check to login endpoint`
- `refactored: extracted validation into separate module`
- `added: new user registration flow`
- `configured: CI pipeline with test automation`
- `removed: deprecated authentication middleware`

**UNSAFE (contains trigger keywords):**

- `failed: tests passed` - Contains `failed`
- `error: completed successfully` - Contains `error`
- `blocked: resolved all blockers` - Contains `blocked`
- `aborted: successfully aborted transaction` - Contains `aborted` (ambiguous)
- `rejected: PR accepted after review` - Contains `rejected`

### Rationale

Conditional-blocks dependencies (`bd dep add <blocked> <blocker> --type conditional-blocks`) enable "run B only if A fails" workflows. This is implemented by close-reason keyword matching. If a success reason contains a failure keyword, `bd` incorrectly interprets the task as failed, triggering downstream conditional tasks incorrectly. Always use `bd reason lint` before closing.

## Auditability Protocol

Minimum auditability requirements for every executed task:
- task exists in `bd` before execution
- task is linked to the correct parent outcome
- close path includes verification evidence
- close reason passes `bd reason lint`

For non-routine decisions (ambiguity, override, gate bypass, force-close), append:
- `Decision | Evidence | Risk | Follow-up ID: <decision> | <evidence> | <risk> | <id-or-none>`

For non-hermetic checks, append evidence tuple:
- `Evidence tuple: {ts:<UTC>, env:<env-id>, artifact:<id>}`

## Output Protocol

For claim/close/handoff/intake updates, report:
- `commands | verification (or skipped reason) | state changes | next action`
- include key assumptions and key files consulted when relevant

## Decision Request Template

When blocked >30 minutes or impact >1 hour, use:
- `Decision: <one-line choice>`
- `Blocker: <what cannot proceed>`
- `State: <current behavior + constraints>`
- `Options: A/B(/C) with benefit/cost`
- `Rec: <recommended option + rationale>`
- `Impact: <what changes>`
- `Need: <explicit A/B/C or missing info>`

## Handoff Contract

Always provide:
- commands executed
- verification evidence
- state changes
- next ready items
- next session prompt
- stash status (`none` or restore command)

For blocked/deferred work, include context pack order:
`state; repro; next; files; blockers`

## Operational Guardrails

- Track work in `bd`, not markdown TODO lists.
- Do not manually edit `.beads/issues.jsonl`.
- Prefer non-interactive shell flags (`-f`, `-y`, batch mode).
- Keep one active WIP per actor unless preempted by explicit policy.
- Preserve invariants: external contracts, data integrity, security boundaries.
- Internal refactors may break internal interfaces if invariants are preserved.

## Strict Control Mode

### BD_STRICT_CONTROL

Environment variable for enabling strict control-plane enforcement:

```bash
export BD_STRICT_CONTROL=1
```

**When enabled, the following additional checks apply:**

1. **Anchor Requirement**: `flow claim-next` requires `--require-anchor` and `--parent` by default
2. **Explicit ID Resolution**: Partial IDs must match exactly, no fuzzy matching
3. **Gate Enforcement**: Preflight gates are mandatory, not advisory

**CLI Override:**
- `--strict-control`: Enable strict mode per-command
- `--allow-missing-anchor`: Bypass anchor requirement when explicitly justified

**Rationale:** Strict mode prevents common agent errors in automated workflows where context may be incomplete.

## Standing Policies

These policies are always in effect:

1. **WIP Policy**: One in_progress issue per actor unless explicitly preempted. Rationale: Focus reduces context switching overhead.

2. **Parallel Rule**: Independent tasks may be claimed by different actors. Rationale: Throughput optimization for non-conflicting work.

3. **Stickiness + Anchors**: Tasks remain in module scope via anchor labels. Rationale: Context coherence within bounded areas.

4. **Defer Policy**: Deferred issues MUST have `--until` date. Rationale: Deferred without until = hidden forever.

5. **External-Blocker Caveat**: External blockers need context pack with expected resolution. Rationale: External deps are invisible without explicit tracking.

6. **Context Pack Format**: `state; repro; next; files; blockers`. Rationale: Canonical order enables quick context recovery.

7. **Parent-Close Check**: Cannot close parent with open children. Rationale: Incomplete decomposition = incomplete outcome.

8. **Context Freshness**: Context pack must be updated when state changes. Rationale: Stale context = wrong decisions.

9. **Living State Digest**: Anchor issues include state digest in notes. Rationale: Quick resume without full history scan.

10. **Multi-Agent Overlap**: Explicit coordination required when work intersects. Rationale: Race conditions corrupt state.

11. **Commit Granularity**: One logical change per commit, linked to issue ID. Rationale: Atomic commits enable atomic rollbacks.

## Canonical References

- Split orchestrator: `claude-plugin/agents/task-agent.md`
- Split roles:
  - `claude-plugin/agents/beads-query-agent.md`
  - `claude-plugin/agents/beads-issue-manager-agent.md`
  - `claude-plugin/agents/beads-execution-coordinator-agent.md`
  - `claude-plugin/agents/beads-cleanup-agent.md`
