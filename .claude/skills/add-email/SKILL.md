---
name: add-email
description: Adds a generic IMAP/SMTP email channel to NanoClaw for email-native participation (threads + mailing lists). Use when implementing, installing, or discussing the `/add-email` skill (distinct from `/add-gmail`).
---

# `/add-email` (IMAP/SMTP) — Email channel skill

This skill adds **generic email** as a NanoClaw channel: **IMAP** for inbound (ImapFlow) and **SMTP** for outbound (Nodemailer). It is **provider-agnostic** (works with any IMAP/SMTP server) and is **not** `/add-gmail` (Gmail API + OAuth). 

It's designed to allow NanoClaw to participate in trusted email groups. 

## Status

Planning and development for the email channel lives on **`westarete/nanoclaw`** branch **`skill/email`** until merged upstream.

## Caveats

This skill is **not yet fully implemented** in upstream NanoClaw. When it lands, installation will follow the normal feature-skill pattern: merge the **`skill/email`** branch from the appropriate remote (West Arete fork initially, upstream later), then configure IMAP/SMTP credentials via OneCLI and env.

## Development

Read these files if you're working on development of `add-email` skill itself.

- **Implementation plan**: [`PLAN.md`](PLAN.md) (phases, checklists, and email-specific security boundaries)
- **Contributor workflow + principles**: [`DEVELOPMENT.md`](DEVELOPMENT.md) (fork/upstream flow, security mindset, and Skill authoring best practices)
