# ADR-001: OpenAB Loop — Closed-Loop Agent Automation

- **Status:** Proposed
- **Date:** 2026-06-09
- **Author:** Pahud Hsieh / OpenAB Team

## Context

For the past two years, multi-agent systems have been driven one prompt at a time — a human issues a task, an agent executes, the human reviews, and manually triggers the next step. This linear model creates bottlenecks at every handoff point.

The industry is moving toward **agent looping**: a pattern where agents autonomously cycle through discovery → planning → execution → verification until a goal is met, without requiring human intervention at each step.

OpenAB already has the primitives for a loop:
- Agent identities and role boundaries
- Discord-based inter-agent communication
- GitHub webhook integration
- A structured PR review spec with severity levels

What we lack is the **automatic iteration mechanism** — the wiring that lets agents hand off to each other and repeat cycles without human triggering.

## Decision

We will implement a **closed-loop PR review flow** as the first OpenAB Loop, using the existing communication infrastructure (Discord mentions + GitHub webhooks) without modifying the OAB core.

### Loop Design

```
PR opened → webhook → Reviewer Agent → verdict
                                          │
                          ┌───────────────┴───────────────┐
                          │                               │
                     LGTM ✅                    CHANGES REQUESTED ⚠️
                          │                               │
                     notify human                  dispatch to Coder
                     (await merge)                 (Discord mention)
                                                          │
                                                   Coder fixes → push
                                                          │
                                                   GitHub webhook
                                                   (synchronize)
                                                          │
                                                   Reviewer re-review
                                                          │
                                                   (loop back to verdict)
```

### Trigger Chain

| Step | Trigger | Actor | Mechanism |
|------|---------|-------|-----------|
| 1 | PR opened / synchronize | Loop Controller | GitHub webhook (existing) |
| 2 | Review complete, verdict = CHANGES_REQUESTED | Reviewer Agent | Discord mention to Coder (new) |
| 3 | Fix complete | Coder Agent | `git push` (existing) |
| 4 | New commits on PR | GitHub | webhook `synchronize` event (existing) |

Only **Step 2 is new**. All other steps use existing infrastructure.

### Dispatch Format

When Reviewer posts `CHANGES REQUESTED`, it sends a structured Discord message mentioning the Coder agent:

```json
{
  "action": "fix",
  "pr": 42,
  "repo": "openab-io/openab",
  "branch": "feat/example",
  "iteration": 1,
  "max_iterations": 3,
  "findings": [
    {
      "id": 1,
      "severity": "🔴",
      "file": "src/auth.rs",
      "line": 23,
      "issue": "missing input validation",
      "suggestion": "add bounds check"
    }
  ]
}
```

### Coder Agent Behavior on Dispatch

1. Verify sender is in the allowed dispatcher list
2. Check `iteration < max_iterations`, otherwise escalate to human
3. Check findings do not involve security/auth/infra, otherwise escalate
4. Checkout branch, apply fixes for 🔴 (must fix) then 🟡 (should fix)
5. Run tests — if fail, escalate to human
6. Push — GitHub webhook automatically triggers re-review

### Safety Boundaries (Hard Stop Conditions)

- `iteration >= max_iterations` (default: 3)
- Findings involve security / auth / infrastructure changes
- Same finding unresolved for 2 consecutive iterations
- Tests fail after fix
- Human explicitly intervenes ("stop" / "I'll handle it")

Human override has the highest priority at all times.

## Open vs Closed Looping

| | Open Loop | Closed Loop |
|---|---|---|
| Nature | Exploratory, wide solution space | Bounded, predefined path |
| Pros | Can discover unexpected solutions | Cheap, controllable, improves each run |
| Cons | Burns tokens, risk of low-quality output | Requires upfront path design |
| Use for | Architecture exploration, new features | Routine work, repeatable flows |

This ADR implements a **closed loop**. Open loops may be explored in future ADRs once token budgets and model capabilities mature.

## Implementation Phases

1. **Phase 1 (MVP):** Reviewer dispatches to Coder via Discord mention; Coder fixes and pushes; webhook triggers re-review. No new infra required.
2. **Phase 2:** Add eval gates — structured quality checks at each step beyond just "does it build."
3. **Phase 3:** Loop history and feedback accumulation — each completed loop feeds context to future runs.
4. **Phase 4:** Configurable loop parameters in `config.toml` (max_iterations, auto_merge, severity filters, escalation rules).

## Consequences

### Positive
- Reduces human bottleneck in PR review cycles
- Leverages existing infrastructure (Discord, webhooks) — no core changes needed
- Human retains full visibility and override capability via Discord
- Each iteration improves agent behavior through accumulated feedback

### Negative
- Risk of infinite loops if safety boundaries fail — mitigated by hard iteration cap
- Token cost increases with iteration count — bounded by max_iterations
- Agents may produce low-quality fixes under pressure — mitigated by eval gates (Phase 2)

### Neutral
- Does not require changes to OAB core
- Does not require user-facing config changes in Phase 1
- Compatible with future open-loop exploration

## References

- Agent looping concept (industry pattern, 2025-2026)
- OpenAB communication protocol (`shared/communication-protocol.md`)
- OpenAB PR review spec (`shared/pr-review-spec.md`)
