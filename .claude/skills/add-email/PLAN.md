# `/add-email` — Implementation plan (IMAP/SMTP email channel)

Build a new NanoClaw channel skill (`/add-email`) that makes email a first-class channel via **IMAP** (inbound) + **SMTP** (outbound). 

## Contents

- Scope and posture
- Phase 0: Decisions to lock
- Phase 1: MVP channel (real short-term deliverables)
- Phase 2: Hardening + ops
- Roadmap (first guess)
- Appendix: reference links and prior art

## Scope and posture

- **Name**: `/add-email` (capability), not `/add-imap` (mechanism).
- **Not `/add-gmail`**: Gmail skill is Gmail API + OAuth and a “background resource” pattern; `/add-email` is email-native participation via IMAP/SMTP.
- **Security posture**: security is architectural, not a late “docs pass”. The email channel is a trust boundary that must enforce structural guarantees (credentials, recipient routing, allowlists).

## Phase 0 — Lock the key decisions

These decisions unblock implementation and prevent thrash.

### 0.1 Thread addressing model (how `sendMessage(jid, text)` can thread email)

NanoClaw’s interface leans “group chat”: `sendMessage(jid, text)`. Email needs (at least) recipients + subject + threading headers.

- [ ] Decide the **JID scheme** for email groups and per-thread routing.\n  - Baseline: group JID is `email:<address>` (e.g. `email:flying@list.org`).\n  - Decide how thread identity is stored/resolved for outbound replies.
- [ ] Decide the **reply context source of truth**.\n  - Minimal: channel persists last-seen thread per group (SQLite).\n  - Ensure scheduled tasks can reply to the right “current thread” (see 0.3).
- [ ] Decide how to represent thread metadata in SQLite (minimal table is fine for v1).\n  - Must include: `message_id`, `in_reply_to`, `references`, normalized subject, timestamps, and mapping back to group JID.

### 0.2 Routing model (direct vs mailing list)

One IMAP account can receive:

- Direct mail to the bot identity (reply to sender)
- Mailing list traffic (participate in the list thread)

- [ ] Decide how to detect **direct vs list** delivery.\n  - Parse `To`, `Cc`, and `List-Id` (and optionally `Delivered-To` / `X-Original-To` depending on provider).\n  - Decide the mapping to NanoClaw groups (one group per list address is the default mental model).
- [ ] Decide whether “one IMAP connection feeds multiple NanoClaw groups” is **Phase 1** (if required) or **Roadmap** (if not required immediately).

### 0.3 Scheduled outbound threading (“start new thread” vs “reply to current”)

Example: a Wednesday email starts a new thread; Thu/Fri reply to that same thread.

- [ ] Decide how the channel chooses between:\n  - **StartNewThread** (compose new outbound)\n  - **ReplyToThread** (threaded reply)
- [ ] Decide where the policy lives:\n  - Channel config (host-side, structural)\n  - Scheduler/task metadata (host-side)\n  - Avoid asking the container agent to “format headers” as the primary mechanism.

## Phase 1 — MVP email channel (short-term, realistic)

Goal: A minimal, robust email channel that can receive messages, store them in NanoClaw, and send correctly threaded replies, with structural security boundaries.

### 1.1 Channel skeleton + registration

- [ ] Add a new channel implementation (expected path: `src/channels/email.ts`).
- [ ] Self-register channel in `src/channels/index.ts` (side-effect import), matching the existing channel pattern.
- [ ] Implement the Channel interface methods NanoClaw expects:\n  - `connect()` / `disconnect()`\n  - `isConnected()`\n  - `ownsJid()` (recognize `email:` scheme)\n  - `sendMessage(jid, text)` (thread-aware)\n  - Optional: `syncGroups()` (Phase 2 if not required for MVP)

### 1.2 Inbound IMAP receive (IDLE-first)

- [ ] Use **ImapFlow** for IMAP connection + mailbox locking.
- [ ] Support IMAP **IDLE** (push) with reconnect handling.\n  - Clear behavior on disconnect: backoff + rebuild client; avoid wedging.
- [ ] On new mail:\n  - Fetch envelope + required headers (`Message-ID`, `In-Reply-To`, `References`, `List-Id` when present)\n  - Extract text content from `text/plain` or fall back from HTML.\n  - Flatten into NanoClaw’s `NewMessage` shape (map sender, sender_name, content, timestamps; thread fields when available).

### 1.3 Thread metadata persistence (SQLite side table)

- [ ] Create minimal persistence for threading so restarts don’t lose context.\n  - Store the headers you need to produce correct replies (`inReplyTo`, `references`, subject) keyed by group JID + thread identity.
- [ ] Define a deterministic “current thread” policy per group for v1.\n  - Example: most recent inbound message’s thread wins unless overridden by scheduled-task policy.

### 1.4 Outbound SMTP send (threaded replies)

- [ ] Use **Nodemailer** transport created once and reused.
- [ ] Implement `sendMessage(jid, text)`:\n  - Resolve recipients (structural; agent does not choose addresses).\n  - Choose reply vs new thread (Phase 0.3 decision).\n  - Set `inReplyTo`, `references`, and `subject` so Gmail-style clients thread reliably.
- [ ] Add an outbound queue (minimal) if connection can be temporarily unavailable.\n  - Phase 2 can make this more robust; Phase 1 just shouldn’t drop replies silently.

### 1.5 Email-specific security boundaries (MVP requirements)

These are structural constraints; do not rely on prompting the model.

- [ ] **Credentials never enter the container**.\n  - IMAP/SMTP auth is read by the host-side NanoClaw process (via OneCLI) and used only in the channel.\n  - Container agent should only ever see message text + safe metadata.
- [ ] **Inbound allowlist (optional but supported)**.\n  - Unknown senders can be dropped before reaching orchestrator/agent.
- [ ] **Outbound allowlist (optional but supported)**.\n  - Channel resolves recipients; validate against allowlist before sending.\n  - Agent cannot override this (because it never specifies recipients).

### 1.6 Tests (minimum credible coverage)

- [ ] Unit tests for:\n  - Header parsing (`Message-ID`, `In-Reply-To`, `References`, `List-Id`)\n  - Direct-vs-list detection\n  - Thread persistence and outbound threading decisions\n  - Allowlist enforcement (inbound + outbound)
- [ ] A small “dry run” harness (optional) to simulate inbound mail objects without real IMAP during tests.

## Phase 2 — Hardening and operations

- [ ] Reconnect strategy: backoff, jitter, and clear logging (avoid hot loops).
- [ ] Outbound queue: persistence across restart (if needed).
- [ ] Multi-group routing from one IMAP account (if not done in Phase 1):\n  - map list headers / recipient addresses to NanoClaw group JIDs deterministically.
- [ ] `syncGroups()` behavior:\n  - optionally expose discovered lists (from `List-Id` / `To`) as group metadata for UX.\n  - keep it best-effort and safe.
- [ ] Observability:\n  - log key events (connect/disconnect, fetch errors, send failures)\n  - metrics counters if NanoClaw has a pattern for this

## Roadmap (first guess)

- **Attachments**: safe handling + explicit allowlist of MIME types; avoid auto-downloading untrusted binaries into sensitive contexts.
- **Calendar invites**: parse `.ics`, extract text summary, and optionally surface structured fields.
- **Better thread model**: multiple active threads per group; explicit selection for scheduled tasks; admin tooling.
- **Provider quirks**: Gmail labels / ProtonMail Bridge / self-hosted TLS edge cases.
- **Marketplace + upstream**: publish a community marketplace entry; open upstream PR when Phase 1 is stable.

## Appendix — Reference links and prior art

### Libraries

- **ImapFlow** (IMAP receive): `https://imapflow.com/docs/guides/basic-usage`
- **Nodemailer** (SMTP send): `https://nodemailer.com/usage/`
- **html-to-text** (HTML fallback): `https://github.com/html-to-text/node-html-to-text`

### NanoClaw contracts (in this repo)

- Channel + message types: `src/types.ts`
- Channel registration: `src/channels/registry.ts`, `src/channels/index.ts`
- Inbound message handling: `src/index.ts` (`onMessage`, allowlist patterns)
- Outbound routing: `src/router.ts` (`routeOutbound`)

### Upstream channel patterns to follow (implementation style)

- Telegram: `qwibitai/nanoclaw-telegram` (threading + reply context patterns)
- Slack: `qwibitai/nanoclaw-slack` (group participation + reconnect/queue patterns)

### Community prior art (operational patterns only)

- OpenClaw IMAP skill (node-imap stack, not ImapFlow): `https://github.com/openclaw/skills/tree/main/skills/mvarrieur/imap-email`

