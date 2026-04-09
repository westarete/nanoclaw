# Development notes — `/add-email` (West Arete fork)

This document is for **developing** the `/add-email` skill (not for end-user “how to install” usage).

## Mandatory reading (for Claude working on this skill)

Before doing any significant edits to this skill’s docs or implementation, always read and follow:

- Anthropic: [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

Local expectations derived from that doc:

- Keep `SKILL.md` **short** and use **progressive disclosure** (point to one-level-deep references like `PLAN.md` and `DEVELOPMENT.md`).
- Avoid “deep nesting” where `SKILL.md` points to a doc that points to another doc for core requirements.
- Use checklists/workflows when sequences are fragile (connect/reconnect loops, migration steps, security invariants).

## Introduction to NanoClaw architecture

If you're new to NanoClaw, start with [`intro-to-nanoclaw-architecture.md`](intro-to-nanoclaw-architecture.md) for a tour of the concepts this skill builds on (channels, registry, routing, containers, groups, and memory boundaries).

## Repository roles (West Arete context)

How this repository relates to upstream NanoClaw and other repos. These are conventions for maintainers of this fork, not rules for upstream.

| Repo | Role |
|------|------|
| [`qwibitai/nanoclaw`](https://github.com/qwibitai/nanoclaw) | Upstream — source of truth for core. Add as remote `upstream`. |
| This fork ([`westarete/nanoclaw`](https://github.com/westarete/nanoclaw)) | West Arete — public fork where we develop skills (e.g. `/add-email`), `.claude/skills/add-email/`, and open PRs upstream. `origin` points here. |

Use separate clones per org when it keeps remotes and commit identity obvious; avoid juggling multiple unrelated `origin` remotes in one directory.

## Branch model

- `main` — Tracks `upstream/main`. Prefer it fast-forwardable to upstream: merge `upstream/main` into `main` regularly.
- `skill/<name>` (e.g. `skill/email`) — Feature work following NanoClaw’s skills-as-branches model. Channel implementation and `.claude/skills/add-email/` live here.

## Staying up to date

1. `git fetch upstream`
2. `git checkout main` → `git merge upstream/main`
3. `git checkout skill/email` → `git merge main` (or `git merge upstream/main`)

Merge is the default once a branch is pushed/shared. Rebase is fine for solo work before a PR if you want a linear story; avoid rebasing branches others depend on.

## Pull requests to upstream

- One logical change per PR.
- Keep the PR branch reasonably current with `upstream/main` (merge or rebase as appropriate).
- Reviewers focus on the PR diff (scope, tests, `CONTRIBUTING.md`), not your fork’s `main` topology.

## Security principles (operator + maintainer mindset)

These are broader NanoClaw principles that should shape this skill’s implementation and defaults.

### Break the main group’s [ABC] chain (Agents Rule of Two)

The main group can violate Meta’s “Agents Rule of Two” by combining:

- [A] untrustworthy inputs (web content)
- [B] access to sensitive systems
- [C] state-changing capability

That combination is an avoidable risk: prompt injection can propagate across memory boundaries (e.g. global memory) and across groups.

Options (choose structural guarantees over “please behave”):

- Remove [A]: keep the privileged/admin group free of untrusted inputs (no WebFetch/WebSearch there).
- Remove [C]: make global memory structurally read-only to containers (mount read-only).
- Remove [B]: sandbox browsing away from sensitive data and admin operations (one-way “lockdown” once untrusted content enters).

### Establish a security mindset in docs and architecture

- NanoClaw owns its security model; dependency safety is defense-in-depth, not the strategy.
- Threat model the architecture you control: where untrusted input enters, persists, and can change state.
- Prefer structural controls (mount perms, tool availability, code paths) over behavioral prompting.

### Default toward an admin/personal split

Avoid using the most privileged chat context as the most exposed browsing/research context.

- **Admin**: privileged operations only, no untrusted inputs. (`isMain: true` lives here.)
- **Personal**: daily driver with browsing/research, but no cross-group admin power.

### Identity disambiguation in groups

When a channel provides insufficiently unique display names (e.g. only first name), disambiguate `sender_name` so the agent can track participants reliably.

## Open questions (development)

### Threading implementation

See [`nanoclaw-threading.md`](nanoclaw-threading.md) for the available threading approaches in NanoClaw. Our best option is probably to leverage (and upstream) the top recommendation there, and join the thread.

### Testing strategy

- What is the MVP test approach?\n  - Unit tests around parsing + threading + allowlists (fast, deterministic)\n  - An optional “integration-ish” harness that can run against a real IMAP/SMTP account in a manual/dev mode\n  - Decide what (if anything) should run in CI vs only locally

### Scope: channel vs container-side tools (compare `/add-gmail`)

`/add-gmail` effectively ships two things:\n- a **channel** (host-side integration that receives/sends)\n- **tools** (container-side MCP server that lets the agent search/read/draft/send)\n\nOpen decision for `/add-email`:\n- Is `/add-email` strictly a **channel** (v1), relying on NanoClaw’s normal message loop + scheduler?\n- Or do we also ship **container-side tools** for richer email operations (search, drafts, multi-thread selection), and if so: when, and with what security boundaries?
