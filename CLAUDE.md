# CLAUDE.md — ZeroClaw

## Commands

```bash
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test
```

Full pre-PR validation (recommended):

```bash
./dev/ci.sh all
```

Docs-only changes: run markdown lint and link-integrity checks. If touching bootstrap scripts: `bash -n install.sh`.

## Project Snapshot

ZeroClaw has two roles:

1. **Agent runtime** — a Rust-first autonomous agent runtime optimized for performance, efficiency, stability, extensibility, sustainability, and security.
2. **Spec orchestration server** — an event-driven coordination server for spec-driven development, where contributors submit specs (not code), autonomous agents certify/execute/validate against those specs, and humans act as judges for conflicts.

Core architecture is trait-driven and modular. Extend by implementing traits and registering in factory modules. The project dogfoods its own spec-driven protocol.

### Spec-Driven Development Protocol

- Contributors submit specs, not code. Agents certify specs as unambiguous before execution.
- Agents get N attempts before escalating with learnings. PR validation diffs code against specs.
- Spec-vs-code conflicts escalate to human judges. Contributors bring their own compute and agent stack.

Spec format: YAML frontmatter + markdown body. Each spec has a unique atomic ID (`{domain}-{NNN}`, e.g., `auth-003`). IDs belong to atomic requirements, not files. Lifecycle: `draft → certified → active → amended → active`. Code links back via `// SPEC(id)` markers with bidirectional validation.

Ticket lifecycle: `open → certifying → ready → assigned → executing → review → merged/failed`.

### Spec Linter (`zeroclaw-spec-lint`)

Validates frontmatter structure, ID uniqueness, and path existence. Cross-references `// SPEC()` markers in code against spec `implements` lists. Rebuilds `specs/_index.yaml` manifest. Commands: `check`, `index`, `ci`.

### Event-Driven Server

Reacts to: spec created/amended, ticket certified, PR opened, agent attempt failed, conflict detected, human judgment received. Initiates: agent assignments, certification requests, lint runs, escalation notifications, auto-ticket creation.

### Extension Points

- `src/providers/traits.rs` (`Provider`)
- `src/channels/traits.rs` (`Channel`)
- `src/tools/traits.rs` (`Tool`)
- `src/memory/traits.rs` (`Memory`)
- `src/observability/traits.rs` (`Observer`)
- `src/runtime/traits.rs` (`RuntimeAdapter`)
- `src/peripherals/traits.rs` (`Peripheral`) — hardware boards (STM32, RPi GPIO)
- `specs/` directory with `_index.yaml` manifest
- Spec linter binary (`zeroclaw-spec-lint`)

## Repository Map

- `src/main.rs` — CLI entrypoint and command routing
- `src/lib.rs` — module exports and shared command enums
- `src/config/` — schema + config loading/merging
- `src/agent/` — orchestration loop
- `src/gateway/` — webhook/gateway server
- `src/security/` — policy, pairing, secret store
- `src/memory/` — markdown/sqlite memory backends + embeddings/vector merge
- `src/providers/` — model providers and resilient wrapper
- `src/channels/` — Telegram/Discord/Slack/etc channels
- `src/tools/` — tool execution surface (shell, file, memory, browser)
- `src/peripherals/` — hardware peripherals (STM32, RPi GPIO)
- `src/runtime/` — runtime adapters (currently native)
- `specs/` — spec files organized by domain, with `_index.yaml` manifest
- `docs/` — topic-based documentation (setup-guides, reference, ops, security, hardware, contributing, maintainers)
- `.github/` — CI, templates, automation workflows

## Risk Tiers

- **Low risk**: docs/chore/tests-only changes
- **Medium risk**: most `src/**` behavior changes without boundary/security impact
- **High risk**: `src/security/**`, `src/runtime/**`, `src/gateway/**`, `src/tools/**`, `.github/workflows/**`, access-control boundaries, `specs/` structure changes, spec linter rules, orchestration event routing

When uncertain, classify as higher risk.

## Workflow

1. **Read before write** — inspect existing module, factory wiring, and adjacent tests before editing.
2. **One concern per PR** — avoid mixed feature+refactor+infra patches.
3. **Implement minimal patch** — no speculative abstractions, no config keys without a concrete use case.
4. **Validate by risk tier** — docs-only: lightweight checks. Code changes: full relevant checks.
5. **Document impact** — update PR notes for behavior, risk, side effects, and rollback.
6. **Queue hygiene** — stacked PR: declare `Depends on #...`. Replacing old PR: declare `Supersedes #...`.

Branch/commit/PR rules:
- Work from a non-`master` branch. Open a PR to `master`; do not push directly.
- Use conventional commit titles. Prefer small PRs (`size: XS/S/M`).
- Follow `.github/pull_request_template.md` fully.
- Never commit secrets, personal data, or real identity information (see `@docs/contributing/pr-discipline.md`).

## Anti-Patterns

- Do not add heavy dependencies for minor convenience.
- Do not silently weaken security policy or access constraints.
- Do not add speculative config/feature flags "just in case".
- Do not mix massive formatting-only changes with functional changes.
- Do not modify unrelated modules "while here".
- Do not bypass failing checks without explicit explanation.
- Do not hide behavior-changing side effects in refactor commits.
- Do not include personal identity or sensitive information in test data, examples, docs, or commits.

## Linked References

- `@docs/contributing/change-playbooks.md` — adding providers, channels, tools, peripherals; security/gateway changes; architecture boundaries
- `@docs/contributing/pr-discipline.md` — privacy rules, superseded-PR attribution/templates, handoff template
- `@docs/contributing/docs-contract.md` — docs system contract, i18n rules, locale parity
