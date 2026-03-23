# Swarm Protocol: Spec-Driven Agent Development System

## Overview

A protocol for open source projects where contributors submit **intent** (tickets + specs), not code. Autonomous agents generate, validate, and certify all artifacts. Humans act as judges, not laborers.

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
├── specs/              # source of truth
│   ├── core/
│   ├── integrations/
│   └── protocol/
├── tickets/            # active ticket queue
│   ├── pending/
│   ├── certified/
│   ├── in_progress/
│   └── escalated/
├── src/                # generated + core runtime (minimal)
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
