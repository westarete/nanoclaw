# `/add-email` — skill documentation

Entry point for [West Arete](https://westarete.com)’s [NanoClaw](https://github.com/qwibitai/nanoclaw) fork work on the **IMAP/SMTP email channel** skill. Active development lives on the [`skill/email`](https://github.com/westarete/nanoclaw/tree/skill/email) branch. Session context for agents: [CLAUDE.md](CLAUDE.md); the repo root [CLAUDE.md](../../../CLAUDE.md) lists this path under Key Files.

Bookmark on GitHub: [`.claude/skills/add-email/README.md` on `skill/email`](https://github.com/westarete/nanoclaw/blob/skill/email/.claude/skills/add-email/README.md)

The default `main` branch tracks upstream. Its [root README](https://github.com/westarete/nanoclaw/blob/main/README.md) starts with a short West Arete fork notice and link here; the rest matches [upstream’s README](https://github.com/qwibitai/nanoclaw/blob/main/README.md).

This directory holds the **feature skill** materials for `/add-email` (planning, fork workflow, security notes) — not core upstream NanoClaw docs. See [SKILL.md](SKILL.md) for the slash command entry point.

## NanoClaw skill model

Feature skills, `skill/*` branches, merges, and marketplaces are documented once in upstream’s [Skills as branches](https://github.com/qwibitai/nanoclaw/blob/main/docs/skills-as-branches.md) guide.

## Documents

| Doc | Purpose |
|-----|---------|
| [CLAUDE.md](CLAUDE.md) | Agent/session context for this skill (what to read, SKILL vs planning docs) |
| [PLAN.md](PLAN.md) | Primary technical plan (IMAP/SMTP email channel) |
| [DEVELOPMENT.md](DEVELOPMENT.md) | How this fork relates to upstream, branches, PRs, other repos |
| [security-principles.md](security-principles.md) | Security posture for extending and operating this fork |
