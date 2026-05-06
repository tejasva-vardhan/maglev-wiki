**Status:** Proposed

**Owner:** Eric

---

## Context

Maglev is a volunteer-driven project. Contributors come and go, and the team has no shared institutional memory beyond the code itself, informal discussion in GitHub issues, PRs, and slack threads. This makes it hard for new contributors to understand why things are the way they are, and makes revisiting past decisions risky, it's easy to either blindly preserve something that should change, or blindly reverse something that was carefully considered.

At the same time, we're a small volunteer team. A heavyweight RFC or design doc process would be more burden than benefit.

## Decision

We will adopt a lightweight Architecture Decision Record (ADR) process, based on the format proposed by Michael Nygard: https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions

ADRs will be stored on the wiki, numbered sequentially (e.g. `ADR-0001: Adopt Architecture Decision Records`).

Each ADR will follow this structure:

- **Title** — a short noun phrase
- **Status** — one of: Proposed, Accepted, Deprecated, Superseded
- **Owner** — the contributor proposing and shepherding this ADR
- **Context** — the forces and circumstances motivating the decision, written neutrally
- **Decision** — what we decided, written in active voice ("We will…")
- **Consequences** — the resulting tradeoffs, positive and negative

Each ADR has a designated **owner**: the contributor who researches the problem, writes the document, and drives the conversation.

An ADR moves from *Proposed* to *Accepted* by [[rough|https://datatracker.ietf.org/doc/html/rfc7282#page-7]] consensus among active contributors. If a later decision reverses or replaces an ADR, the original is marked *Superseded* with a reference to its replacement.

ADRs should be one or two pages.

ADRs take a broad view of what counts as architecture. Beyond high-level design, this includes how the software is decomposed into modules, how processes are structured and communicate at runtime, how components are mapped to infrastructure, and how the codebase itself is organised and developed, including CI/CD pipelines and the metrics we use to measure development health. If a decision has a meaningful effect on any architectural view point, it's worth an ADR.

## Consequences

New contributors will have a navigable record of why the project is structured and run the way it is. Revisiting past decisions becomes safer because the original context is preserved. Contributors gain a concrete, bounded way to drive a piece of the project.

The process only works if ADRs are actually written. There is a risk that decisions get made informally and never recorded. We should treat "this feels architecturally significant" as a prompt to open an ADR.
