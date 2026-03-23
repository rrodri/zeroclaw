# Swarm Protocol: Spec-Driven Agent Development System

## Overview

A protocol for open source projects where contributors submit **intent** (tickets + specs), not code. Autonomous agents generate, validate, and certify all artifacts. Humans act as judges, not laborers.

**This is a standalone event-driven coordination server** — not a feature of any single agent runtime. It orchestrates agents (ZeroClaw, Devin, Cursor, local LLMs, anything) by owning the spec lifecycle, ticket state machine, and routing logic. The server reacts to events (spec amended, PR opened, agent failed) and initiates actions (assign agent, escalate, create ticket). Agents are workers that speak the protocol; the server is the control plane.

---

## Core Principles

1. **Specs are law** — all generated code is validated against specs, never the reverse
2. **Agents are labor** — any agent stack (Claude, Cursor, local LLM) can participate
3. **CI is enforcement** — the only authority on whether output meets spec
4. **Humans are judges** — intervene only to file intent and resolve ambiguity
5. **Failed work adds value** — every failed attempt enriches the ticket for the next agent

---

## Architecture

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌────────────┐
│  Ticket      │────▶│  Certification   │────▶│  Execution       │────▶│  PR        │
│  Submission  │     │  Gate            │     │  (3 attempts)    │     │  Validation│
└─────────────┘     └──────────────────┘     └──────────────────┘     └────────────┘
       │                    │                        │                       │
       │              ┌─────┴─────┐            ┌─────┴─────┐          ┌─────┴─────┐
       │              │ Agent A   │            │ Executing │          │ Validator │
       │              │ Agent B   │            │ Agent     │          │ Agents    │
       │              │ (agree?)  │            │ (any stack│          │ (diff vs  │
       │              └───────────┘            │  BYO)     │          │  spec)    │
       │                    │                  └───────────┘          └───────────┘
       │                    │                        │                       │
       │              [reject: open               [fail x3:              [violation:
       │               questions back              escalate w/            escalate to
       │               to submitter]               learnings]             human judge]
       │                                                                    │
       │                                                              ┌─────┴─────┐
       │                                                              │  Human    │
       │                                                              │  Decision │
       │                                                              │           │
       │                                                              │ Update    │
       │                                                              │ spec? or  │
       │                                                              │ update PR?│
       │                                                              └───────────┘
       │
  [dogfood: framework processes its own tickets]
```

---

## Ticket Schema

```yaml
ticket:
  id: string              # deterministic hash of (title + spec_ref + created_by)
  title: string
  spec_ref: string         # path to spec file this ticket targets
  description: string      # what should change and why
  acceptance_criteria:     # machine-verifiable conditions
    - criterion: string
      verification: "test" | "lint" | "spec_match" | "ci_pass"
  constraints:             # boundaries the agent must not cross
    - string
  context:                 # accumulated across attempts
    attempts: int          # 0-3
    history:
      - attempt: int
        agent_id: string
        model: string
        tokens_spent: int
        duration_ms: int
        outcome: "success" | "fail"
        artifacts: [string]  # PR refs, branch names
        failure_reason: string | null
        learnings: string | null  # what the next agent should know
  certification:
    status: "pending" | "certified" | "rejected" | "needs_clarification"
    agent_a: { id: string, verdict: "pass" | "fail", questions: [string] }
    agent_b: { id: string, verdict: "pass" | "fail", questions: [string] }
  escalation:
    tier: "standard" | "elevated" | "human_required"
    reason: string | null
  cost:
    total_tokens: int
    total_duration_ms: int
```

---

## Spec Format

Specs are the source of truth. All validation references them.

### Atomic IDs

Each spec has a unique atomic ID at the **requirement** level, not the file level. One ID = one independently implementable, independently verifiable requirement. Format: `{domain}-{NNN}` (e.g., `auth-003`, `mem-017`, `gateway-002`). A single file may group multiple specs, but each has its own ID and lifecycle.

The test for atomicity: if you can change requirement A without logically affecting requirement B, they get separate IDs.

### Spec Lifecycle

```
draft → certified → active → amended → active
```

| State | Meaning |
|-------|---------|
| `draft` | Written, not yet reviewed by certification agents |
| `certified` | Two independent agents agree the spec is unambiguous |
| `active` | Implementation exists and passes validation |
| `amended` | Spec changed intentionally; implementation is now stale (auto-ticket created) |

When a spec moves from `active` to `amended`, the system auto-creates an implementation ticket. Code linked to that spec is flagged as **stale** (not divergent). A TTL/SLA window prevents false-positive CI failures while agents catch up.

**Divergence vs. staleness:**

| Situation | Detection | Action |
|-----------|-----------|--------|
| Code changed, spec didn't | Agent diffs PR against spec | Block PR, escalate |
| Spec changed, code didn't | Spec state = `amended` | Auto-create ticket, no block |
| Both changed independently | Conflict | Escalate to human judge |

### Language-Native Spec Shadow Tree

Specs are written as **compilable interface files in the same language as the source**, not in YAML or a custom DSL. Every source file can have a spec mirror using the `.spec` extension:

```
src/                          specs/
  auth/                         auth/
    token.rs          ↔           token.rs.spec
    session.rs        ↔           session.rs.spec
  gateway/                      gateway/
    webhook.rs        ↔           webhook.rs.spec
```

The `.spec` file contains types, signatures, constants, and doc-comment constraints — no implementation bodies. The native compiler validates structural conformance.

**Rust** (`token.rs.spec`):
```rust
/// SPEC(auth-003): Token refresh retry policy
///
/// When a token refresh request fails with a transient error,
/// the system retries the request.
pub trait TokenRefreshSpec {
    /// Retries exactly 3 times
    /// Backoff: exponential, base=500ms, max=4s
    const MAX_RETRIES: u32 = 3;
    const BASE_DELAY_MS: u64 = 500;
    const MAX_DELAY_MS: u64 = 4000;

    /// On exhaustion, returns AuthError::RefreshExhausted
    fn refresh_token(&self, token: &ExpiredToken) -> Result<Token, AuthError>;

    /// Applies to transient errors only
    fn is_transient(err: &AuthError) -> bool;

    /// Does NOT apply to 4xx client errors
    fn is_client_error(err: &AuthError) -> bool;
}
```

**Java** (`FooService.java.spec`):
```java
/// SPEC(foo-001): Foo processing pipeline
public interface FooServiceSpec {
    static final int MAX_BATCH_SIZE = 100;
    static final Duration TIMEOUT = Duration.ofSeconds(30);
    CompletableFuture<FooResult> process(FooRequest request) throws FooException;
}
```

**TypeScript** (`userStore.ts.spec`):
```typescript
/// SPEC(user-001): User persistence contract
export interface UserStoreSpec {
    readonly MAX_CONNECTIONS: 10;
    get(id: UserId): Promise<User | null>;
    save(user: User): Promise<void>;
    // Must not: delete users, only soft-delete
    softDelete(id: UserId): Promise<void>;
}
```

**Python** (`processor.py.spec`):
```python
## SPEC(proc-001): Event processing contract
class ProcessorSpec(Protocol):
    MAX_QUEUE_DEPTH: int = 1000
    FLUSH_INTERVAL_MS: int = 5000
    def process(self, event: Event) -> ProcessResult: ...
    def flush(self) -> None: ...
```

**Go** (`handler.go.spec`):
```go
/// SPEC(gw-002): Request handler contract
type HandlerSpec interface {
    MaxBodyBytes() int64    // must return 1_048_576
    Handle(ctx context.Context, req *Request) (*Response, error)
    // Must not: panic on malformed input
}
```

**Why language-native specs:**
- The spec compiles with the same toolchain — no new parser needed
- `impl Trait` / `implements Interface` IS the bidirectional link (no `// SPEC(id)` markers needed in source code when using traits/interfaces directly)
- The compiler rejects structural spec violations at build time
- Every developer already knows how to write a spec — it's just an interface in their language

### Verification Pyramid

```
        ▲
       / \        Doc comments → agent-generated behavior tests
      /   \       (runtime verification)
     /─────\
    /       \     Associated constants → assertion tests
   /         \    (compile-time values, runtime assertions)
  /───────────\
 /             \  Trait/interface signatures → native compiler
/               \ (compile-time structural verification)
───────────────────
```

Bottom layer is free (compiler). Middle layer is mechanical (constants are directly testable). Top layer needs an agent, but doc comments are scoped to a single method, minimizing ambiguity.

### Constraint Patterns in Doc Comments

Doc comments use a small vocabulary of structured natural-language constraint patterns that linters can parse and agents can verify:

```
- count: exactly N / at most N / at least N
- timing: within Xms / after Xms / every Xms
- on [event]: [action]
- returns: [type or value]
- calls: [function/service] with [params]
- must not: [negative constraint]
- applies to: [scope]
- does not apply to: [exclusion]
```

The ambiguity linter flags weasel words:
```
- retries: a few times          ← FAIL: "a few" is ambiguous
- retries: exactly 3            ← PASS
- timeout: reasonably fast      ← FAIL: "reasonably" is ambiguous
- timeout: within 5000ms        ← PASS
```

### Spec Index

A `specs/_index.yaml` manifest is auto-generated by the linter on every pass:

```yaml
specs:
  - id: auth-003
    file: specs/auth/token.rs.spec
    status: active
    language: rust
    implements: [src/auth/token.rs, src/providers/retry.rs]
  - id: auth-004
    file: specs/auth/session.rs.spec
    status: amended
    language: rust
    implements: [src/auth/session.rs]
```

Agents use the index for fast lookup. The linter regenerates it and fails CI if the committed version doesn't match.

### Legacy YAML Spec Format

For non-code specs (protocol definitions, architecture constraints, cross-cutting concerns), the YAML format remains available:

```yaml
spec:
  id: string
  name: string
  version: int              # incremented on every human-approved change
  description: string

  interface:                # what the component exposes
    inputs:
      - name: string
        type: string
        required: bool
    outputs:
      - name: string
        type: string

  behavior:                 # what it must do
    - given: string
      when: string
      then: string

  constraints:              # what it must not do
    - string

  dependencies:             # other specs this relies on
    - spec_ref: string
      version: ">= N"

  validation_rules:         # machine-enforceable rules checked on every PR
    - rule: string
      check: "regex" | "ast" | "test" | "llm_review"
      target: string        # file glob or module path

  history:                  # audit trail
    - version: int
      changed_by: string    # human or "escalation from ticket X"
      reason: string
      diff: string
```

---

## Pipeline Stages

### Stage 1: Ticket Submission

- Submitter (human or agent) creates ticket referencing a spec
- Ticket ID = hash(title + spec_ref + created_by) for dedup
- Ticket enters certification queue
- Cost tracking initialized at zero

### Stage 2: Certification Gate

Two independent agents review the ticket (not the code — the ticket itself).

**Each agent evaluates:**
- Are acceptance criteria machine-verifiable?
- Is the spec reference valid and current?
- Are there ambiguities that would cause two competent agents to produce different solutions?
- Are constraints clear?

**Outcomes:**
| Agent A | Agent B | Result |
|---------|---------|--------|
| pass    | pass    | certified → execution queue |
| pass    | fail    | fail questions merged → back to submitter |
| fail    | pass    | fail questions merged → back to submitter |
| fail    | fail    | rejected with combined questions |

**Certification agents must be different model families or configurations to avoid correlated blind spots.**

### Stage 3: Execution

Any agent can claim a certified ticket. The protocol doesn't care what stack runs it.

**Per attempt:**
1. Agent pulls ticket + spec + any prior attempt history
2. Generates artifacts (code, tests, config)
3. Runs local validation (tests, lint, spec match)
4. Submits PR if validation passes
5. Attempt logged to ticket regardless of outcome

**Attempt budget: 3.** After 3 failures:
- Ticket re-enters queue with `tier: elevated`
- All failure context (learnings, errors, partial solutions) preserved
- Higher-tier model or human required to unblock

**Cost tracked per attempt.** Enables decisions like "this ticket has burned 200K tokens, worth a human look."

### Stage 4: PR Validation

Validation agents diff the commit range against the referenced spec.

**Checks:**
1. **Spec compliance** — does the diff satisfy every `behavior` rule?
2. **Constraint adherence** — does the diff violate any `constraints`?
3. **Scope creep** — does the diff modify files outside the spec's `target` globs?
4. **Regression** — do existing spec tests still pass?
5. **Validation rules** — all `validation_rules` in the spec pass (regex, AST, test, LLM review)?

**On violation:**
```
violation:
  spec_ref: string
  rule_violated: string
  evidence: string          # the specific diff hunk or test failure
  suggestion: string        # what the agent thinks should change
```

Escalated to human. Human picks one:
- **Update the spec** → spec version incremented, audit trail logged, all future PRs validated against new version
- **Update the PR** → ticket goes back to execution with violation context

### Stage 5: Merge

PR passes validation → auto-merge or human approval (configurable per-repo policy).

---

## Escalation Tiers

| Tier | Trigger | Handler |
|------|---------|---------|
| standard | new certified ticket | any contributor agent |
| elevated | 3 failed attempts | higher-capability model |
| human_required | spec-vs-code conflict, repeated elevation failures | human judge |

Tier transitions are automatic. Each escalation carries full context from prior tiers.

---

## Contributor Model

### What contributors bring
- **Compute**: their own machine, their own API keys, their own agent
- **Tickets**: problem descriptions and specs
- **Judgment**: spec-vs-code conflict resolution

### What contributors don't need
- Knowledge of the codebase internals
- Ability to write code in the project's language
- Understanding of the full architecture

### Contribution flow
```
1. git clone the-project
2. Browse ticket queue (or file a new ticket)
3. Point their agent at a certified ticket
4. Agent works locally on their hardware
5. Agent opens PR
6. CI + validation agents handle the rest
```

---

## Dogfooding

The framework processes its own tickets using this protocol.

- New integration needed? File a spec + ticket.
- Bug found? File a ticket referencing the violated spec.
- Spec wrong? Human updates it, triggers re-validation of affected code.
- Framework generator produces bad output? That's a bug ticket against the generator spec.

The repository is:
```
repo/
├── specs/              # source of truth — language-native .spec files
│   ├── auth/
│   │   ├── token.rs.spec
│   │   └── session.rs.spec
│   ├── gateway/
│   │   └── webhook.rs.spec
│   ├── protocol/       # YAML specs for cross-cutting concerns
│   └── _index.yaml     # auto-generated manifest
├── tickets/            # active ticket queue
│   ├── pending/
│   ├── certified/
│   ├── in_progress/
│   └── escalated/
├── src/                # implementation (must satisfy specs/)
├── tests/              # spec-derived test suites
└── .protocol/          # protocol config
    ├── config.yaml     # escalation rules, attempt limits, tier policies
    ├── agents.yaml     # registered certification/validation agents
    └── history/        # audit log of all spec changes and escalations
```

---

## Protocol Interface

Any agent interacts via these operations:

```
CLAIM(ticket_id) → ticket + spec + history
SUBMIT(ticket_id, commit_range, artifacts) → validation_result
CERTIFY(ticket_id, verdict, questions?) → certification_status
VALIDATE(pr_id, spec_ref, commit_range) → [violations]
ESCALATE(ticket_id, reason, learnings) → new_tier
UPDATE_SPEC(spec_id, diff, reason) → new_version  # human only
```

Transport is irrelevant. Git-native (tickets as files in repo), REST API, message queue — the protocol doesn't care. Agents speak the operations, not the transport.

---

## Spec Linter (`zeroclaw-spec-lint`)

A Rust binary that validates the spec shadow tree. Three passes:

1. **Parse** — walk `specs/`, detect language from extension, validate structure
2. **Cross-ref** — compile `.spec` files with native compilers, verify source files implement the spec interfaces
3. **Index** — rebuild `specs/_index.yaml`, diff against committed version

**Commands:**
```bash
zeroclaw-spec-lint check                      # full validation
zeroclaw-spec-lint check specs/auth/token.rs.spec  # single spec
zeroclaw-spec-lint index                      # rebuild index
zeroclaw-spec-lint ci                         # index + check + nonzero exit on violation
```

**Structural checks (per spec file):**

| Rule | Check |
|------|-------|
| ID matches heading | `/// SPEC(auth-003):` must appear in the spec file |
| ID is unique | No two spec files share an ID |
| ID format | Must match `^[a-z]+-\d{3,}$` |
| Status is valid | One of `draft`, `certified`, `active`, `amended` |
| Implements paths exist | Every path in the index `implements` list is a real file |
| Dependencies exist | Every referenced spec ID exists |
| Spec compiles | Language-native compiler succeeds on the `.spec` file |

**Cross-reference checks (spec ↔ source):**

| Rule | Check |
|------|-------|
| Source satisfies spec | Source file implements the trait/interface defined in `.spec` |
| No orphan specs | If `status: active`, at least one source file implements it |
| No weasel words | Doc comments don't contain "should", "might", "usually", "sometimes" |
| Constants are exact | No `approximately`, no ranges where exact values work |

**GitHub Action sensor (thin glue):**
A lightweight Action runs `zeroclaw-spec-lint ci` on PRs touching `specs/` or source files with spec implementations, posts the result as a check, and notifies the server.

---

## Event-Driven Coordination Server

The server is a **long-lived process** that owns the spec lifecycle, ticket state machine, and agent routing. It is not a CI runner — it maintains persistent state and holds connections to agents.

### Events the server reacts to

| Event | Source | Action |
|-------|--------|--------|
| Spec created | Git push to `specs/` | Validate, set `draft`, queue for certification |
| Spec amended | Git push modifying active spec | Set `amended`, auto-create implementation ticket |
| Ticket certified | Certification agents report | Move ticket to execution queue |
| PR opened | GitHub webhook | Run spec validation, flag violations |
| Agent attempt failed | Agent reports via protocol | Decrement budget, reassign or escalate |
| Spec-vs-code conflict | Validation agents detect | Route to human judge |
| Human judgment received | Dashboard/API | Update spec or reject PR, resume pipeline |
| TTL expired on amended spec | Internal timer | Escalate stale implementation ticket |

### Actions the server initiates

- Agent assignments (broadcast ticket to available agents)
- Certification requests (ask N agents to review a spec)
- Lint/validation runs (trigger `zeroclaw-spec-lint ci`)
- Escalation notifications (human judge needed — Slack/Discord/email)
- Auto-ticket creation (when a spec is amended)
- Grace period management (TTL on amended specs)

### State the server owns

| Entity | States |
|--------|--------|
| Spec | `draft → certified → active → amended → active` |
| Ticket | `open → certifying → ready → assigned → executing → review → merged/failed` |
| Attempt | `running → passed/failed` (max N per ticket) |
| Escalation | `pending → judged` |
| Agent | `available → busy → reporting` |

### Deployment

The server runs as a standalone process (VPS, container, self-hosted). It needs:
- A stable address for GitHub webhooks
- Outbound HTTPS for GitHub API, agent communication, notifications
- Persistent storage for state (SQLite/Postgres)

```
GitHub ──webhooks──▶ ┌─────────────────┐ ──assign──▶ Agent A (ZeroClaw)
                     │                 │ ──assign──▶ Agent B (any runtime)
Specs  ──push────▶   │    Server       │ ──certify─▶ Agent C
                     │                 │
Humans ──judge───▶   │  (state +       │ ──lint───▶  Spec linter
                     │   events +      │
Agents ──report──▶   │   routing)      │ ──notify──▶ Slack/Discord/etc
                     └─────────────────┘
```

---

## Open Source Strategy

The protocol and tooling are open. The server is closed.

**Open source:**

| Component | Reason |
|-----------|--------|
| Spec format (`.spec` convention) | It's a standard — standards die closed |
| Spec linter (`zeroclaw-spec-lint`) | Adoption driver — teams use it before needing the server |
| Constraint patterns vocabulary | Must be in everyone's codebase |
| Agent protocol (CLAIM/SUBMIT/etc) | BYO compute only works if the protocol is public |
| GitHub Action sensor | Thin glue, no value closed |

**Closed source:**

| Component | Reason |
|-----------|--------|
| Orchestration server | This is the product — state machine, routing, queue management |
| Agent assignment/scheduling | Optimization logic, competitive edge |
| Escalation routing + UX | Human judge experience is the differentiator |
| Analytics/observability | "Which specs cause the most failures" — insight teams pay for |
| Multi-tenant hosting | Managed service revenue |

**The linter is the trojan horse.** Teams add `zeroclaw-spec-lint` to CI → their specs follow the format → at 20 specs and 5 agents they need the server → already in their pipeline.

---

## Security Considerations

- **Agents never see other contributors' API keys** — all execution is local
- **Specs are append-only versioned** — no silent overwrites, full audit trail
- **Certification requires independent agents** — prevents single-agent rubber-stamping
- **Commit ranges are validated, not trusted** — malicious PRs caught by spec validation
- **Escalation to human is mandatory for spec changes** — agents cannot rewrite the rules
- **Ticket hashing prevents replay/duplication** — deterministic IDs from content

---

## Cost Model

Every operation logs tokens and wall time. Aggregated per ticket:

```
ticket X:
  certification: 4K tokens, 2 agents
  attempt 1: 45K tokens, failed (missing edge case)
  attempt 2: 52K tokens, failed (test timeout)
  attempt 3: 38K tokens, success
  validation: 8K tokens
  total: 147K tokens
```

Enables:
- Estimating ticket difficulty before claiming
- Identifying specs that consistently produce expensive tickets (spec needs refinement)
- Contributors deciding which tickets are worth their compute budget
- Project-level metrics on automation efficiency

---

## What This Is Not

- **Not a distributed compute platform** — no shared compute, no trust model needed
- **Not a CI system** — CI is a dependency, not a replacement
- **Not an agent framework** — it's a protocol. BYO agent.
- **Not a code review tool** — validation is spec compliance, not style/quality opinions
