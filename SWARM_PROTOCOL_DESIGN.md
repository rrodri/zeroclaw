# Swarm Protocol: Spec-Driven Agent Development System

## Overview

A protocol for open source projects where contributors submit **intent** (tickets + specs), not code. Autonomous agents generate, validate, and certify all artifacts. Humans act as judges, not laborers.

**This is a standalone event-driven coordination server.** It orchestrates headless coding agents (Devin, Factory, Codegen, Claude Code, Cursor, local LLMs — anything with an API) by owning the spec lifecycle, ticket state machine, and routing logic. The server reacts to events (spec amended, PR opened, agent failed) and initiates actions (assign agent, escalate, create ticket). Agents are workers that speak the protocol; the server is the control plane.

**This project does not include an agent runtime.** The agent market is commoditizing — every IDE and AI lab is shipping headless coding agents. Building another one is a losing race. The coordination layer has zero real competitors. We build the dispatch, not the worker.

---

## Core Principles

1. **Specs are law** — all generated code is validated against specs, never the reverse
2. **Agents are labor** — any headless coding agent (Devin, Factory, Codegen, Claude Code, Cursor, local LLM) can participate via adapters
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

### Universal Spec Language

Specs are written in a **language-agnostic format** inspired by Gherkin's readability and math textbook rigor. One format works regardless of whether the implementation is Rust, Go, Python, TypeScript, or anything else. Agents bridge the gap by generating language-specific code and tests from universal specs.

**Design principles:**
- Gherkin's `Given/Then` for human readability (most readable structured language, 20+ years of BDD validation)
- Math textbook `Define` blocks for scoped, referenceable terms (no undefined variables, no contradictions)
- Fixed constraint vocabulary for machine-parseable precision
- Breakable cross-references that the linter validates

**Example:**

```spec
spec auth-003 "Token refresh retry policy"
status active
version 3

Define
  transient_error: 5xx, timeout, connection reset
  client_error: 4xx, malformed request
  max_retries: 3
  base_delay: 500ms
  max_delay: 4s
  backoff_strategy: exponential

Given token refresh fails with {transient_error}
Then retry the request
  count: exactly {max_retries}
  backoff: {backoff_strategy}
  base_delay: {base_delay}
  max_delay: {max_delay}
  order: sequential

Given retries reach {max_retries}
Then return {err-001:RefreshExhausted}
  log at error

Given each retry attempt
Then log at warn
  includes: attempt number, delay duration, error type

Given token refresh fails with {client_error}
Then fail immediately
  count: exactly 0
  log at error
  includes: status code, response body

Boundary
  applies to: {transient_error}
  does not apply to: {client_error}
  does not apply to: initial authentication (only refresh)

Implements
  src/auth/token.rs
  src/providers/retry.rs

Depends on
  net-012 >= 3
  err-001
```

**A spec it depends on:**

```spec
spec err-001 "Error type registry"
status active
version 5

Define
  auth_errors: RefreshExhausted, TokenExpired, Unauthorized, Forbidden
  network_errors: Timeout, ConnectionReset, DnsFailure
  severity_levels: recoverable, terminal

Given an error is {auth_errors}
Then classify as domain error
  severity: terminal

Given an error is {network_errors}
Then classify as infrastructure error
  severity: recoverable

Given an unknown error
Then classify as infrastructure error
  severity: terminal
  log at error
  includes: raw error message, stack context

Boundary
  applies to: all errors surfaced to callers
  does not apply to: internal retry logic (retries handle their own errors)

Implements
  src/errors/types.rs
  src/errors/classify.rs
```

### Grammar

The entire grammar fits on a napkin. Parsed by [pest](https://pest.rs) (PEG parser generator for Rust — grammar lives in a `.pest` file, zero-copy parsing, great error messages).

```
spec       := "spec" ID TITLE
status     := "status" STATE
version    := "version" INT
define     := "Define" "\n" (INDENT NAME ":" VALUE)+
rule       := "Given" TEXT "\n" "Then" TEXT "\n" constraint*
constraint := INDENT KEY ":" VALUE
ref        := "{" NAME "}" | "{" SPEC-ID ":" SYMBOL "}"
boundary   := "Boundary" "\n" scope+
scope      := INDENT ("applies to:" | "does not apply to:") TEXT
implements := "Implements" "\n" filepath+
depends    := "Depends on" "\n" (SPEC-ID (">=" INT)?)+
```

~10 production rules. ~200 lines of Rust for the parser.

### Reference System

References are the key differentiator from Gherkin (which has no reference system at all).

**Local references** — terms defined in the same spec:
```spec
Define
  max_retries: 3

Given retries reach {max_retries}    ← resolves to 3
```

**Cross-spec references** — symbols from another spec:
```spec
Then return {err-001:RefreshExhausted}   ← err-001 must exist AND define RefreshExhausted
```

**Version-pinned dependencies:**
```spec
Depends on
  net-012 >= 3     ← net-012 must exist at version 3 or higher
```

**What breaks and when:**

| Reference type | Breaks when | Linter error |
|---|---|---|
| `{term}` | Term not in `Define` block | `auth-003: undefined reference {max_retires} (typo?)` |
| `{spec-id:symbol}` | Spec doesn't exist or doesn't define symbol | `auth-003: {err-001:RefreshExhausted} — err-001 has no such define` |
| `Implements` path | File deleted, moved, renamed | `auth-003: implements target src/auth/token.rs not found` |
| `Depends on` spec ID | Spec deleted or ID changed | `auth-003: depends on net-012 but no spec with that ID exists` |
| Version pin | Dependency version too low | `auth-003: depends on net-012 >= 3 but net-012 is version 2` |
| Circular dependency | `auth-003 → net-012 → auth-003` | `auth-003: circular dependency detected` |
| Duplicate define | Same name defined twice | `auth-003: duplicate define "timeout"` |

### Constraint Vocabulary

A fixed set of ~20-30 constraint types. Anything outside this vocabulary is a linter error — forces spec authors to be precise.

```
- count:      exactly N / at most N / at least N
- timing:     within Xms / after Xms / every Xms
- backoff:    exponential / linear / fixed
- base_delay: duration
- max_delay:  duration
- size:       at most 1MB / exactly 4096 bytes
- order:      sequential / parallel / any
- on failure: return ErrorType / retry / skip / escalate
- severity:   recoverable / terminal
- log at:     debug / info / warn / error
- includes:   comma-separated list of what to include
- returns:    type or value
- calls:      function/service with params
- must not:   negative constraint
```

**Weasel word detection:**
```
- retries: a few times          ← FAIL: "a few" is ambiguous
- retries: exactly 3            ← PASS
- timeout: reasonably fast      ← FAIL: "reasonably" is ambiguous
- timeout: within 5000ms        ← PASS
```

### How Agents Use Specs

An agent reads the universal `.spec` file and generates language-specific code + tests. The spec is the same regardless of target language:

**From the spec:**
```spec
Given retries reach {max_retries}
Then return {err-001:RefreshExhausted}
```

**Agent generates (Rust):**
```rust
#[test]
fn auth_003_exhaustion_returns_correct_error() {
    let refresher = TokenRefresher::new(mock_always_fail());
    let result = refresher.refresh_token(&expired_token());
    assert!(matches!(result, Err(AuthError::RefreshExhausted)));
}
```

**Agent generates (TypeScript):**
```typescript
test('auth-003: exhaustion returns RefreshExhausted', async () => {
  const refresher = new TokenRefresher(alwaysFailProvider);
  await expect(refresher.refresh()).rejects.toThrow(RefreshExhaustedError);
});
```

**Agent generates (Go):**
```go
func TestAuth003_ExhaustionReturnsCorrectError(t *testing.T) {
    refresher := NewTokenRefresher(alwaysFailProvider)
    _, err := refresher.RefreshToken(expiredToken)
    assert.ErrorIs(t, err, ErrRefreshExhausted)
}
```

### Spec Index

A `specs/_index.yaml` manifest is auto-generated by the linter on every pass:

```yaml
specs:
  - id: auth-003
    file: specs/auth/auth-003.spec
    status: active
    version: 3
    implements: [src/auth/token.rs, src/providers/retry.rs]
    depends_on: [net-012, err-001]
  - id: err-001
    file: specs/errors/err-001.spec
    status: active
    version: 5
    implements: [src/errors/types.rs, src/errors/classify.rs]
    depends_on: []
```

Agents use the index for fast lookup. The linter regenerates it and fails CI if the committed version doesn't match.

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
├── specs/              # source of truth — universal .spec files
│   ├── auth/
│   │   ├── auth-003.spec
│   │   └── auth-004.spec
│   ├── gateway/
│   │   ├── gw-001.spec
│   │   └── gw-002.spec
│   ├── errors/
│   │   └── err-001.spec
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

### Agent Adapters

Headless coding agents (Devin, Factory, Codegen, Claude Code, etc.) each have their own API. The server doesn't talk to them directly — it talks through **adapters** that translate between the protocol operations and each agent's native API.

```
Server ──CLAIM──▶ ┌─────────────┐ ──Devin API──▶ Devin
                  │  Adapter     │
Server ──SUBMIT─▶ │  (per agent) │ ──Factory API──▶ Factory
                  │              │
Server ◀─REPORT── │  Translates  │ ──Claude CLI──▶ Claude Code
                  │  protocol ↔  │
                  │  native API  │ ──Local exec──▶ Local LLM
                  └─────────────┘
```

Each adapter implements a single interface:

```go
type AgentAdapter interface {
    // Dispatch a certified ticket to the agent
    Assign(ctx context.Context, ticket Ticket, spec Spec) (AttemptID, error)

    // Poll or receive status updates
    Status(ctx context.Context, attemptID AttemptID) (AttemptStatus, error)

    // Retrieve artifacts (branch, PR, logs) when attempt completes
    Collect(ctx context.Context, attemptID AttemptID) (Artifacts, error)

    // Cancel a running attempt
    Cancel(ctx context.Context, attemptID AttemptID) error
}
```

**Shipping adapters:**

| Adapter | Connects to | How |
|---------|-------------|-----|
| `devin` | Devin API | REST — create session, attach spec, poll for PR |
| `factory` | Factory.ai API | REST — submit task, receive webhook on completion |
| `codegen` | Codegen API | REST — create task from spec |
| `claude-cli` | Claude Code CLI | Local exec — spawn process, pass spec as prompt, collect git output |
| `cursor` | Cursor headless | Local exec — workspace + spec injection |
| `local-llm` | Ollama / vLLM / llama.cpp | Local exec — prompt with spec, run in sandboxed workspace |
| `custom` | Any agent | Webhook — server posts ticket, agent posts back results |

**Adapters are open source.** They're thin glue — 100-300 lines each. Community contributes new ones. The more adapters exist, the more agents the server can dispatch to, the more valuable the platform.

**The `custom` webhook adapter is the escape hatch.** Any agent that can receive an HTTP POST and return results speaks the protocol. No SDK needed.

---

## Spec Linter (`zeroclaw-spec-lint`)

A Rust binary (using pest for parsing) that validates the universal spec format. Three passes:

1. **Parse** — walk `specs/`, parse each `.spec` file against the pest grammar, validate structure
2. **Cross-ref** — resolve all `{term}`, `{spec-id:symbol}`, `Implements`, and `Depends on` references across the spec tree
3. **Index** — rebuild `specs/_index.yaml`, diff against committed version

**Commands:**
```bash
zeroclaw-spec-lint check                          # full validation
zeroclaw-spec-lint check specs/auth/auth-003.spec # single spec
zeroclaw-spec-lint index                          # rebuild index
zeroclaw-spec-lint ci                             # index + check + nonzero exit on violation
```

**Structural checks (per spec file):**

| Rule | Check |
|------|-------|
| Grammar valid | File parses against pest grammar |
| ID format | Must match `^[a-z]+-\d{3,}$` |
| ID unique | No two spec files share an ID |
| Status valid | One of `draft`, `certified`, `active`, `amended` |
| Has boundary | At least one `applies to` or `does not apply to` |
| Has title | Quoted string present |
| Constraints known | Every constraint key is in the vocabulary |
| Defines used | Every `Define` entry is referenced (warning if unused) |
| No duplicate defines | Same name not defined twice |

**Reference checks (cross-spec):**

| Rule | Check |
|------|-------|
| Local refs resolve | Every `{term}` has a matching `Define` entry |
| Cross refs resolve | Every `{spec-id:symbol}` points to an existing spec with that define |
| Implements paths exist | Every file path in `Implements` is a real file |
| Depends on specs exist | Every spec ID in `Depends on` has a `.spec` file |
| Version pins satisfied | `Depends on net-012 >= 3` fails if net-012 is version 2 |
| No circular dependencies | Dependency graph is acyclic |
| No orphan specs | If `status: active`, `Implements` is non-empty |
| No weasel words | Given/Then/constraint text doesn't contain "should", "might", "usually", "approximately", "sometimes", "reasonably" |

**Example linter output:**
```
$ zeroclaw-spec-lint check

specs/auth/auth-003.spec
  ✓ Grammar valid
  ✓ ID format valid
  ✓ All defines used
  ✓ All references resolve
  ✓ {err-001:RefreshExhausted} → err-001 defines RefreshExhausted
  ✓ Implements: src/auth/token.rs exists
  ✓ Implements: src/providers/retry.rs exists
  ✓ Depends on: err-001 exists (version 5, need >= 1)
  ✓ Depends on: net-012 exists (version 3, need >= 3)
  ✓ Boundary present
  ✓ No weasel words

specs/errors/err-001.spec
  ✓ Grammar valid
  ✓ ID format valid
  ✓ All defines used
  ✓ All references resolve
  ✓ Implements: src/errors/types.rs exists
  ✓ Implements: src/errors/classify.rs exists
  ✓ No dependencies
  ✓ Boundary present
  ✓ No weasel words

2 specs, 0 errors, 0 warnings
```

**Error messages (from pest):**
```
specs/auth/auth-003.spec:14:3
  |
14|   count: approximately 3
  |          ^^^^^^^^^^^^^
  = expected duration, integer, or size
```

**GitHub Action sensor (thin glue):**
A lightweight Action runs `zeroclaw-spec-lint ci` on PRs touching `specs/` or source files listed in `Implements`, posts the result as a check, and notifies the coordination server.

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
                     ┌─────────────────┐
GitHub ──webhooks──▶ │                 │ ──adapter──▶ Devin
                     │                 │ ──adapter──▶ Factory
Specs  ──push────▶   │    Server       │ ──adapter──▶ Codegen
                     │    (Go+Dapr)    │ ──adapter──▶ Claude Code
Humans ──judge───▶   │                 │ ──adapter──▶ Local LLM
                     │  state machine  │
Agents ──report──▶   │  ticket actors  │ ──lint───▶  Spec linter
                     │  adapter pool   │
                     │                 │ ──notify──▶ Slack/Discord/etc
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
| Agent adapters | Community contributes new ones — more adapters = more agents = more value |
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

- **Not an agent runtime** — the agent market is commoditizing. We don't build workers, we dispatch them.
- **Not a distributed compute platform** — no shared compute, no trust model needed
- **Not a CI system** — CI is a dependency, not a replacement
- **Not an agent framework** — it's a protocol + coordination server. BYO agent via adapters.
- **Not a code review tool** — validation is spec compliance, not style/quality opinions

---

## Tech Stack

| Component | Language | Why |
|-----------|----------|-----|
| Coordination server | Go + Dapr | Event-driven, stateful actors (tickets as Dapr actors), first-class Dapr SDK, fast to ship |
| Agent adapters | Go | Same repo as server, thin translation layer per agent API |
| Spec linter | Rust + pest | Parsing performance, pest PEG grammar, `.pest` file IS the spec language documentation |
| Spec language grammar | `.pest` file | Language-agnostic artifact, consumed by the linter |

**Why Go for the server:**
- Dapr SDK is first-class (reference implementation maintained by Microsoft)
- Goroutines for trivial concurrency across agent connections
- stdlib `net/http` covers 90% of server needs — no framework required
- Fast cold start (~5ms), small binary (~10MB)
- Large hiring pool for server/infrastructure work

**What Dapr provides:**

| Capability | Use case |
|------------|----------|
| Pub/sub | Spec events, ticket state changes, agent assignments |
| State store | Ticket queue, spec index, attempt ledger |
| Service invocation | Agent ↔ server communication |
| Bindings | GitHub webhooks, Slack/Discord notifications |
| Actors | Each ticket as a stateful actor with its own lifecycle state machine |
| Observability | Distributed tracing across agents and server |

**The Dapr actor model maps to tickets:**
Each ticket is a Dapr actor. Dapr handles persistence, activation, deactivation, and distribution. The server code only defines state transitions (certify, assign, record attempt, escalate).

**The linter and server don't share a language.** The linter is a CLI binary. The server calls it via exec or exposes it as a Dapr binding. Clean boundary.

---

## Go-To-Market

### Sequence

1. **Ship the linter** (weeks) — open source `zeroclaw-spec-lint`, publish the `.spec` format and pest grammar
2. **Get adoption** (months) — target 50 repos running the linter in CI. Write 3-5 real specs as proof. Record a demo: spec written → linter validates → agent generates code → linter catches violation
3. **Ship the server** (months) — build when teams ask for orchestration. The pull should come from adoption, not push
4. **Monetize** — managed hosting for teams that don't want to operate the server

### Who pays

| Segment | Why they need this |
|---------|-------------------|
| Teams with 10+ agents running | Need orchestration, not more agents |
| Open source projects | Contributors bring agents instead of time, maintainers define specs |
| Regulated industries | Audit trails: spec → certification → implementation → validation |
| Platform engineering teams | Slots into internal developer platforms |

### Defensibility

- Spec format + linter become a standard (network effect)
- More agents speaking the protocol = more valuable server
- Agent-runtime agnostic — doesn't compete with Devin, Factory, Cursor, etc. Coordinates them via adapters
- Dogfooding = own development velocity proves the product

### Risks

- **GitHub builds it into Actions + Copilot.** Window is ~18-24 months. Speed > perfection.
- **Spec writing is hard.** Most teams are bad at it. May need an agent that helps write specs from natural language.
- **Cold start.** Need specs + agents + server before value is visible. Find one team willing to pilot end-to-end.

### Funding path

Ship the linter with adoption traction first. "50 teams run my spec linter in CI and are asking for the server" is a fundable position. A design doc alone is not.
