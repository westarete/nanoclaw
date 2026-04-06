---
name: add-email
description: Add a generic IMAP/SMTP email channel to NanoClaw (West Arete fork — in development on skill/email). Full design in bundled docs; merge the skill branch when the implementation lands.
---

# Add Email Channel (IMAP/SMTP)

This skill adds **generic email** as a NanoClaw channel — IMAP for inbound (ImapFlow), SMTP for outbound (Nodemailer) — distinct from **`/add-gmail`** (Gmail API + OAuth).

**Status:** Planning and docs live in this directory alongside this file. Implementation is developed on **[`westarete/nanoclaw`](https://github.com/westarete/nanoclaw)** branch **`skill/email`** until merged upstream.

## Where to read

| Doc | Purpose |
|-----|---------|
| [README.md](README.md) | Index of fork/skill documentation |
| [PLAN.md](PLAN.md) | Architecture, security model, ImapFlow/Nodemailer, threading, marketplace flow |
| [DEVELOPMENT.md](DEVELOPMENT.md) | Fork vs upstream, branches, PRs |
| [CLAUDE.md](CLAUDE.md) | Agent session context for this skill |
| [security-principles.md](security-principles.md) | Security posture notes |

## Apply (when implemented)

When the channel code exists on the fork, installation will follow the usual feature-skill pattern: merge **`westarete/skill/email`** (or **`upstream/skill/email`** after upstream accepts the PR), then configure IMAP/SMTP credentials via OneCLI and env. Exact steps will be expanded here as the implementation stabilizes.
