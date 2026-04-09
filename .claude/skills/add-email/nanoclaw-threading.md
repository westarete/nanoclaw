# NanoClaw Threading: Recommendations for /add-email

A reference document for developers working on email channel integration and threading support.

---

## The Problem

NanoClaw currently maintains **one Claude Code session per registered group**. The `sessions` table uses `group_folder` as its primary key. All conversations in a group share one session — one conversation history, one context window.

This works for linear channels (WhatsApp, Telegram personal chats) where there is genuinely one conversation happening. It breaks down for channels where multiple independent conversations arrive on the same JID simultaneously.

The field `thread_id` already exists on `NewMessage` in `src/types.ts` but is currently dormant — it is never populated by any channel and never used in routing or session logic.

---

## How Each Channel Currently Handles Threading

### WhatsApp and Telegram (personal chats)
Linear by nature. No threading concept. Reply context is already implemented via `reply_to_message_id`, `reply_to_message_content`, and `reply_to_sender_name` on `NewMessage`. No threading work needed.

### Slack
Slack has first-class threads (replies off a parent message). The current implementation of `/add-slack` **flattens threads** — thread replies arrive as regular channel messages with no thread context. The agent sees all messages interleaved in one session.

This is explicitly documented as a known limitation in `.claude/skills/add-slack/SKILL.md`:

> **Threads are flattened** — Threaded replies are delivered to the agent as regular channel messages. The agent sees them but has no awareness they originated in a thread. Responses always go to the channel, not back into the thread.
>
> Full thread-aware routing (respond in-thread) requires pipeline-wide changes: database schema, `NewMessage` type, `Channel.sendMessage` interface, and routing logic.

### Discord
Discord has first-class threads and forum channels (where every post is a thread). The current implementation is in a **worse state than Slack** — thread messages are silently dropped entirely.

When a message arrives in a Discord thread, `message.channelId` is the thread's own ID, not the parent channel's ID. That JID is never registered, so the message hits the "unregistered channel" guard in `src/channels/discord.ts` and is discarded without logging.

For **forum channels** (where every post is a thread), this means the agent receives zero messages — the channel is completely non-functional for that Discord channel type.

Source: `src/channels/discord.ts` (in the [nanoclaw-discord](https://github.com/qwibitai/nanoclaw-discord) repo)

### Gmail
The `/add-gmail` skill polls the Primary inbox and surfaces emails as messages. There is no thread modeling. Each email is treated as a standalone message. The skill's channel mode is designed for a personal inbox used as a tool, not for group email participation.

### Email (IMAP/SMTP — the goal)
Email is inherently multi-threaded. A Google Group like `flying@cpsa.aero` will have multiple independent threads active simultaneously — trip planning, maintenance discussions, event announcements. These must be kept separate: Victor should maintain independent context per thread, and outbound replies must go back into the correct thread (correct `In-Reply-To` and `References` headers).

---

## Two Approaches in Active Development

### Option A: `sessionContext` primitive — [PR #1583](https://github.com/qwibitai/nanoclaw/pull/1583) by @farooqu

**Design:** Groups opt in with `isolatedSessions: true`. Channels supply an opaque `sessionContext` string on each `NewMessage` (e.g. `thread_ts`, email `Thread-Id`, `userId`). Core derives:

- A **session key** — `${folder}_${sha256(sessionContext).slice(0,12)}` — deterministic, always a valid folder name, used as the Claude session ID key
- A **queue key** — `${chatJid}::ctx::${sessionContext}` — gives each context its own slot in `GroupQueue` so concurrent contexts run in parallel containers without blocking each other

`getMessagesSince` is filtered by `sessionContext` so each context only sees its own message history.

**For `/add-email`:** Set `isolatedSessions: true` on the group, populate `sessionContext` with the email `Thread-Id` header. Threading is handled by core — the channel just supplies the context string.

**Changed files:** `src/types.ts`, `src/db.ts`, `src/container-runner.ts`, `src/index.ts`

**Gap:** No `last_active` timestamp or pruning strategy for stale thread sessions. Old sessions will accumulate indefinitely.

---

### Option B: Virtual JID auto-registration — [PR #1626](https://github.com/qwibitai/nanoclaw/pull/1626) by @rsdrahat

**Design:** Threads become real registered groups with virtual JIDs encoding the thread. For Telegram forum topics: `tg:-1003204831761:t:42`. The channel constructs these from `message_thread_id`; `sendMessage` parses them back to route replies to the correct topic.

Auto-registration creates the group entry, folder, and `CLAUDE.md` on first message in a new topic. Topic containers get the parent group's folder mounted read-only at `/workspace/parent/` for shared memory.

**For `/add-email`:** Would encode thread IDs into email JIDs (e.g. `email:flying@cpsa.aero:t:<thread-id>`), auto-register each thread as a virtual group, and mount the group's shared folder as parent memory.

**Changed files:** `src/channels/telegram.ts`, `src/index.ts`, `src/db.ts`, `src/container-runner.ts`, `src/types.ts`

**Gap:** More complex JID parsing and auto-registration logic. Each thread creates a real DB row and folder, which may be heavyweight at scale.

---

## Comparison

| | PR #1583 (`sessionContext`) | PR #1626 (virtual JIDs) |
|---|---|---|
| Threads as real groups | No — sessions only | Yes — full group per thread |
| Channel code change | Populate `sessionContext` | Construct virtual JIDs, parse on send |
| Concurrent threads | Yes — separate queue slots | Yes — separate groups/containers |
| Message history isolation | Yes — filtered by `sessionContext` | Yes — separate group folders |
| Cross-thread memory | Via group folder (unchanged) | Via parent mount |
| Auto-registration | N/A | Yes — on first message |
| Pruning | Not addressed | Not addressed |
| Complexity | Lower | Higher |

---

## Recommendation for /add-email

**Watch and align with #1583.** It's the more generic primitive — channels supply an opaque context string and core handles the rest. This is exactly right for email: the `Thread-Id` header is a stable, unique identifier per thread and maps cleanly onto `sessionContext`.

**Do not implement a parallel solution.** The `thread_id` field on `NewMessage` and the gap documented in the Slack SKILL.md both indicate the maintainers are aware of this problem. Building a one-off solution in `/add-email` risks diverging from whatever direction core adopts.

**Comment on both PRs** to signal intent. Ask the maintainers which approach they're leaning toward before `/add-email` takes a hard dependency.

---

## The Pruning Gap

Neither PR addresses stale session cleanup. For email, threads can go cold for months. A `last_active` timestamp on sessions would allow periodic pruning:

```sql
-- Suggested addition to whichever approach is adopted
ALTER TABLE ... ADD COLUMN last_active TEXT NOT NULL DEFAULT (datetime('now'));
```

Pruning can run at startup alongside the existing `cleanupOrphans()` call in `src/index.ts`. Worth raising on the chosen PR as a follow-up or including in `/add-email`'s own migration if the core PR doesn't address it.

---

## Also Worth Noting

### Proton Mail — [PR #1570](https://github.com/qwibitai/nanoclaw/pull/1570) by @jorgenclaw

A `/add-proton-email` skill exists in review. It is **not a channel** — it wraps Proton Bridge MCP tools as an agent command interface with an approval gate for outbound email. No inbound listening, no threading. Not relevant to `/add-email`'s architecture but worth being aware of as a parallel effort.

---

## `Channel.sendMessage` Interface Change

Whichever threading approach core adopts, outbound reply-in-thread requires an interface change:

```typescript
// Current
sendMessage(jid: string, text: string): Promise<void>;

// Needed
sendMessage(jid: string, text: string, threadId?: string): Promise<void>;
```

For email, `threadId` maps to the `In-Reply-To` and `References` MIME headers. Without this, the agent can read threads correctly but will always reply top-level, breaking email threading for recipients.

This change is called out explicitly in the Slack SKILL.md known limitations and is a prerequisite for any channel to participate correctly in threaded conversations.
