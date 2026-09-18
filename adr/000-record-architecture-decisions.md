# ADR-000: Record architecture decisions

- Status: Accepted
- Date: 2026-09-17
- Deciders: Mark Miller

## Context

Atlas is a portfolio system. The code will change. The reasoning
behind the shape of the system is what interviews and future-me
actually need. Slack messages and commit bodies are not a design
history.

## Decision

Every cross-cutting choice that is expensive to reverse gets an ADR
in this repository before the matching code lands.

Cheap, local choices stay in service READMEs.

## Consequences

- PRs that change a decision must update or supersede an ADR.
- ADRs are immutable except for status.
- This repo is allowed to exist with no runtime code.
