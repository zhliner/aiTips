---
name: using-agent-skills
description: Discovers and invokes agent skills. Use when starting a session or when you need to discover which skill applies to the current task. This is the meta-skill that governs how all other skills are discovered and invoked.
---

# Using Agent Skills

## Overview

Agent Skills is a collection of engineering workflow skills organized by development phase, plus a handful of cross-cutting orchestration and meta skills (subagent delegation, git worktrees, and OpenCode's own configuration). Each skill encodes a specific process that senior engineers follow. This meta-skill helps you discover and apply the right skill(s) for your current task — including how to pick between skills whose descriptions overlap.

## Skill Discovery

When a task arrives, identify the development phase and apply the corresponding skill:

```
Task arrives
    │
    ├── Working on OpenCode's own config/agents/skills/plugins/MCP? ──→ customize-opencode
    ├── Want to create, edit, or benchmark a skill? ──────────────────→ skill-creator
    │
    ├── Don't know what you want yet? ─────────────────────────────────→ interview-me
    ├── User casually proposes a full feature/concept to build? ───────→ brainstorming
    ├── Have a rough concept, need variants? ───────────────────────────→ idea-refine
    ├── New project/feature/change, need a formal spec? ───────────────→ spec-driven-development
    │
    ├── Have a spec, need a plan + task breakdown? ────────────────────→ planning-and-task-breakdown
    ├── Have a spec, need a self-contained plan doc? ──────────────────→ writing-plans
    │
    ├── Ready to execute a written plan?
    │   ├── Need an isolated workspace first ───────────────────────────→ using-git-worktrees
    │   ├── Separate session, human reviews at checkpoints ────────────→ executing-plans
    │   ├── Current session, one fresh subagent per task ──────────────→ subagent-driven-development
    │   └── 2+ independent tasks, no shared state ──────────────────────→ dispatching-parallel-agents
    │
    ├── Implementing code? ──────────────────────────────────────────────→ incremental-implementation
    │   ├── UI work? ────────────────────────────────────────────────────→ frontend-ui-engineering
    │   ├── API work? ───────────────────────────────────────────────────→ api-and-interface-design
    │   ├── Need better context? ────────────────────────────────────────→ context-engineering
    │   ├── Need doc-verified code? ─────────────────────────────────────→ source-driven-development
    │   └── Stakes high / unfamiliar code? ──────────────────────────────→ doubt-driven-development
    ├── Writing/running tests? ───────────────────────────────────────────→ test-driven-development
    │   └── Browser-based? ──────────────────────────────────────────────→ browser-testing-with-devtools
    ├── Something broke? ─────────────────────────────────────────────────→ debugging-and-error-recovery
    ├── About to claim "done / fixed / passing"? ────────────────────────→ verification-before-completion
    │
    ├── Reviewing code? ──────────────────────────────────────────────────→ code-review-and-quality
    │   ├── Want a dedicated reviewer subagent before merge? ────────────→ requesting-code-review
    │   ├── Too complex? ─────────────────────────────────────────────────→ code-simplification
    │   ├── Security concerns? ───────────────────────────────────────────→ security-and-hardening
    │   └── Performance concerns? ────────────────────────────────────────→ performance-optimization
    │
    ├── Implementation done, tests pass, decide how to integrate? ───────→ finishing-a-development-branch
    ├── Committing/branching? ────────────────────────────────────────────→ git-workflow-and-versioning
    ├── CI/CD pipeline work? ─────────────────────────────────────────────→ ci-cd-and-automation
    ├── Deprecating/migrating? ───────────────────────────────────────────→ deprecation-and-migration
    ├── Writing docs/ADRs? ───────────────────────────────────────────────→ documentation-and-adrs
    ├── Adding logs/metrics/alerts? ──────────────────────────────────────→ observability-and-instrumentation
    └── Deploying/launching? ─────────────────────────────────────────────→ shipping-and-launch
```

## Choosing Between Overlapping Skills

Several skills sound similar. Pick based on these distinctions rather than guessing:

- **brainstorming** vs. **interview-me + idea-refine + spec-driven-development**: `brainstorming` is a single, lightweight conversational loop (understand → ask one question at a time → present a design → get approval) that fits most normal-sized feature requests. Reach for the three-skill chain only when the ask is large enough to need distinct discovery, refinement, and formal-spec phases, or when the user explicitly invokes one of them by name.
- **planning-and-task-breakdown** vs. **writing-plans**: both turn a spec into a plan. `planning-and-task-breakdown` is the deeper process — dependency graph, parallelization analysis, `tasks/plan.md` + `tasks/todo.md` — for work that will span multiple agents or sessions. `writing-plans` is the lighter one: a single self-contained plan document written for an engineer with zero codebase context. Use the former when scope or parallelism is unclear; use the latter when you just need the plan written down before coding starts.
- **executing-plans** vs. **subagent-driven-development** vs. **dispatching-parallel-agents**: all three execute pre-written work via subagents, differing mainly in session/isolation model. `executing-plans` runs in a separate session with human review checkpoints. `subagent-driven-development` dispatches one fresh implementer subagent per task in the current session, with review after each. `dispatching-parallel-agents` is the general primitive for any 2+ independent tasks, plan-related or not. `using-git-worktrees` pairs with any of the three when the work needs a workspace isolated from what you're currently looking at.
- **code-review-and-quality** vs. **requesting-code-review**: `code-review-and-quality` is the review process and checklist itself. `requesting-code-review` is the delegation mechanism — dispatching a reviewer subagent with hand-crafted context so your own session history doesn't leak into the review. Use `requesting-code-review` to invoke `code-review-and-quality` (or any review) from a clean context.
- **verification-before-completion** vs. each skill's own verification step: `verification-before-completion` is the final gate that applies no matter which skill you followed — run it immediately before claiming anything is "done," "fixed," or "passing," even if the skill you used already had its own verification step.
- **customize-opencode** and **skill-creator** sit outside the development lifecycle entirely. Reach for them only when the task is about OpenCode's own configuration (`opencode.json`, agents, skills, plugins, MCP servers, permissions) or about the skills system itself — never for the target codebase's application code.

## Core Operating Behaviors

These behaviors apply at all times, across all skills. They are non-negotiable.

### 1. Surface Assumptions

Before implementing anything non-trivial, explicitly state your assumptions:

```
ASSUMPTIONS I'M MAKING:
1. [assumption about requirements]
2. [assumption about architecture]
3. [assumption about scope]
→ Correct me now or I'll proceed with these.
```

Don't silently fill in ambiguous requirements. The most common failure mode is making wrong assumptions and running with them unchecked. Surface uncertainty early — it's cheaper than rework.

### 2. Manage Confusion Actively

When you encounter inconsistencies, conflicting requirements, or unclear specifications:

1. **STOP.** Do not proceed with a guess.
2. Name the specific confusion.
3. Present the tradeoff or ask the clarifying question.
4. Wait for resolution before continuing.

**Bad:** Silently picking one interpretation and hoping it's right.
**Good:** "I see X in the spec but Y in the existing code. Which takes precedence?"

### 3. Push Back When Warranted

You are not a yes-machine. When an approach has clear problems:

- Point out the issue directly
- Explain the concrete downside (quantify when possible — "this adds ~200ms latency" not "this might be slower")
- Propose an alternative
- Accept the human's decision if they override with full information

Sycophancy is a failure mode. "Of course!" followed by implementing a bad idea helps no one. Honest technical disagreement is more valuable than false agreement.

### 4. Enforce Simplicity

Your natural tendency is to overcomplicate. Actively resist it.

Before finishing any implementation, ask:
- Can this be done in fewer lines?
- Are these abstractions earning their complexity?
- Would a staff engineer look at this and say "why didn't you just..."?

If you build 1000 lines and 100 would suffice, you have failed. Prefer the boring, obvious solution. Cleverness is expensive.

### 5. Maintain Scope Discipline

Touch only what you're asked to touch.

Do NOT:
- Remove comments you don't understand
- "Clean up" code orthogonal to the task
- Refactor adjacent systems as a side effect
- Delete code that seems unused without explicit approval
- Add features not in the spec because they "seem useful"

Your job is surgical precision, not unsolicited renovation.

### 6. Verify, Don't Assume

Every skill includes a verification step. A task is not complete until verification passes. "Seems right" is never sufficient — there must be evidence (passing tests, build output, runtime data).

Per-skill verification is the local check. The project-wide bar that applies to *every* change, regardless of which skill is active, is the Definition of Done: tests pass, no regressions, behavior verified at runtime, docs updated. See `references/definition-of-done.md`. It complements each task's acceptance criteria rather than replacing them.

## Failure Modes to Avoid

These are the subtle errors that look like productivity but create problems:

1. Making wrong assumptions without checking
2. Not managing your own confusion — plowing ahead when lost
3. Not surfacing inconsistencies you notice
4. Not presenting tradeoffs on non-obvious decisions
5. Being sycophantic ("Of course!") to approaches with clear problems
6. Overcomplicating code and APIs
7. Modifying code or comments orthogonal to the task
8. Removing things you don't fully understand
9. Building without a spec because "it's obvious"
10. Skipping verification because "it looks right"

## Skill Rules

1. **Check for an applicable skill before starting work.** Skills encode processes that prevent common mistakes.

2. **Skills are workflows, not suggestions.** Follow the steps in order. Don't skip verification steps.

3. **Multiple skills can apply.** A feature implementation might involve `idea-refine` → `spec-driven-development` → `planning-and-task-breakdown` → `incremental-implementation` → `test-driven-development` → `code-review-and-quality` → `code-simplification` → `shipping-and-launch` in sequence.

4. **When in doubt, start with a spec.** If the task is non-trivial and there's no spec, begin with `spec-driven-development`.

## Lifecycle Sequence

For a complete feature, the typical skill sequence is:

```
1.  interview-me                → Extract what the user actually wants
    (or brainstorming            → lighter single-skill alternative for smaller asks)
2.  idea-refine                 → Refine vague ideas
3.  spec-driven-development     → Define what we're building
4.  planning-and-task-breakdown → Break into verifiable chunks
    (or writing-plans            → a single self-contained plan doc, when that's all you need)
5.  using-git-worktrees         → Isolate the workspace before touching code
6.  context-engineering         → Load the right context
7.  source-driven-development   → Verify against official docs
8.  incremental-implementation  → Build slice by slice
    (delegate via dispatching-parallel-agents / subagent-driven-development / executing-plans
     when tasks are independent or the plan is being executed by subagents)
9.  observability-and-instrumentation → Instrument as you build (runs parallel with 8-10, not after)
10. doubt-driven-development    → Cross-examine non-trivial decisions in-flight
11. test-driven-development     → Prove each slice works
12. verification-before-completion → Confirm with evidence before claiming anything is done
13. code-review-and-quality     → Review before merge
    (dispatch via requesting-code-review for a clean-context reviewer subagent)
14. code-simplification         → Reduce unnecessary complexity while preserving behavior
15. git-workflow-and-versioning → Clean commit history
16. finishing-a-development-branch → Decide how to integrate (merge/PR/cleanup)
17. documentation-and-adrs      → Document decisions
18. deprecation-and-migration   → Retire old systems and move users safely when needed
19. shipping-and-launch         → Deploy safely
```

Not every task needs every skill. A bug fix might only need: `debugging-and-error-recovery` → `test-driven-development` → `verification-before-completion` → `code-review-and-quality`.

`customize-opencode` and `skill-creator` are not part of this sequence — they apply whenever the task is about OpenCode's own configuration or the skills system itself, at whatever point that need arises.

## Quick Reference

| Phase | Skill | One-Line Summary |
|-------|-------|-----------------|
| Define | interview-me | Surface what the user actually wants before any plan, spec, or code exists |
| Define | brainstorming | Conversational explore-design-approve loop before implementing a proposed feature |
| Define | idea-refine | Refine ideas through structured divergent and convergent thinking |
| Define | spec-driven-development | Requirements and acceptance criteria before code |
| Plan | planning-and-task-breakdown | Decompose into small, verifiable tasks |
| Plan | writing-plans | Write a self-contained implementation plan for a zero-context engineer |
| Plan | using-git-worktrees | Isolate feature work in its own workspace before executing a plan |
| Build | incremental-implementation | Thin vertical slices, test each before expanding |
| Build | source-driven-development | Verify against official docs before implementing |
| Build | doubt-driven-development | Adversarial fresh-context review of every non-trivial decision |
| Build | context-engineering | Right context at the right time |
| Build | frontend-ui-engineering | Production-quality UI with accessibility |
| Build | api-and-interface-design | Stable interfaces with clear contracts |
| Build | dispatching-parallel-agents | Delegate 2+ independent tasks to isolated subagents |
| Build | subagent-driven-development | Execute a plan by dispatching one fresh subagent per task |
| Build | executing-plans | Load, critique, and execute a written plan across session checkpoints |
| Verify | test-driven-development | Failing test first, then make it pass |
| Verify | browser-testing-with-devtools | Chrome DevTools MCP for runtime verification |
| Verify | debugging-and-error-recovery | Reproduce → localize → fix → guard |
| Verify | verification-before-completion | Prove completion with evidence before any success claim |
| Review | code-review-and-quality | Five-axis review with quality gates |
| Review | requesting-code-review | Dispatch a reviewer subagent with hand-crafted, history-free context |
| Review | code-simplification | Preserve behavior while reducing unnecessary complexity |
| Review | security-and-hardening | OWASP prevention, input validation, least privilege |
| Review | performance-optimization | Measure first, optimize only what matters |
| Ship | git-workflow-and-versioning | Atomic commits, clean history |
| Ship | finishing-a-development-branch | Present merge/PR/cleanup options once work is done |
| Ship | ci-cd-and-automation | Automated quality gates on every change |
| Ship | deprecation-and-migration | Remove old systems and migrate users safely |
| Ship | documentation-and-adrs | Document the why, not just the what |
| Ship | observability-and-instrumentation | Structured logs, RED metrics, traces, symptom-based alerts |
| Ship | shipping-and-launch | Pre-launch checklist, monitoring, rollback plan |
| Meta | customize-opencode | Edit OpenCode's own config, agents, skills, plugins, MCP servers |
| Meta | skill-creator | Create, edit, and benchmark skills; optimize triggering descriptions |
