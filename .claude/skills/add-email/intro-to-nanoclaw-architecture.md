# NanoClaw Architecture: Groups, Channels, and Agents

A reference document for developers building channel skills and extensions.

---

## What Is a Group?

A group is two things:

1. **A row in the `registered_groups` SQLite table** — maps a chat JID to a folder name, trigger pattern, and flags (`isMain`, `requiresTrigger`, `containerConfig`)
2. **A folder under `groups/`** — the agent's writable workspace, mounted into the container at `/workspace/group`

Groups are registered with a JID (the platform-specific chat identifier), a display name, a folder name, and a trigger word. The folder is where the agent's `CLAUDE.md`, memory files, logs, and any persistent state live.

Key fields in `registered_groups`:

| Field | Purpose |
|---|---|
| `jid` | Platform chat identifier (e.g. `tg:123456`, `dc:987654`, `120363@g.us`) |
| `folder` | Subfolder under `groups/` (e.g. `telegram_main`, `email_flying-cpsa`) |
| `trigger_pattern` | Trigger word (e.g. `@Papa`); ignored if `isMain` or `requiresTrigger=false` |
| `is_main` | Elevated privileges; processes all messages without trigger |
| `requires_trigger` | If false, all messages are processed (used for 1-on-1 chats) |
| `container_config` | JSON: additional mounts, timeout overrides |

Source: `src/db.ts`, `src/types.ts`

---

## What Is a Channel?

A channel implements a thin interface (`src/types.ts`):

```typescript
interface Channel {
  name: string;
  connect(): Promise<void>;
  sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  disconnect(): Promise<void>;
  setTyping?(jid: string, isTyping: boolean): Promise<void>;
  syncGroups?(force: boolean): Promise<void>;
}
```

The critical method is `ownsJid(jid)` — each channel claims JIDs by prefix or pattern:

| Channel | JID pattern |
|---|---|
| Telegram | `tg:*` |
| Discord | `dc:*` |
| WhatsApp | `*@g.us`, `*@s.whatsapp.net` |
| Slack | `slack:*` |

Channels are passive on the inbound side. When a message arrives, the channel calls `onMessage(chatJid, msg)` which stores it in SQLite. The channel does not invoke the agent directly.

Channels self-register at startup via `registerChannel()` in `src/channels/registry.ts`. The barrel file `src/channels/index.ts` imports each installed channel, triggering registration. Channels that lack credentials return `null` from their factory and are silently skipped.

---

## Message Flow: Channel → Agent

```
Channel receives message
  → calls onMessage(chatJid, msg)
    → stored in SQLite (messages table)

Message loop polls SQLite every POLL_INTERVAL ms
  → finds new messages for registered groups
  → checks trigger word (if required)
  → formats messages as XML prompt
  → enqueues to GroupQueue

GroupQueue (serialized per group)
  → processGroupMessages(chatJid)
    → runAgent()
      → runContainerAgent()
        → spawns Docker container
        → passes prompt via stdin as JSON
        → streams output back
          → channel.sendMessage(chatJid, text)
```

Source: `src/index.ts`, `src/container-runner.ts`

---

## What Context Does the Agent Get?

`src/router.ts` formats recent messages as XML:

```xml
<context timezone="America/Los_Angeles" />
<messages>
  <message sender="Alice" time="2:30 PM" reply_to="msg123">
    <quoted_message from="Bob">original content</quoted_message>
    the reply content
  </message>
</messages>
```

This is the `prompt` field sent to the container via stdin. The window is the last `MAX_MESSAGES_PER_PROMPT` messages since the agent last ran for that group. The agent also has its mounted filesystem context (`CLAUDE.md`, group workspace, global memory).

---

## Session Continuity

The `sessions` table stores one `session_id` per `group_folder`. Each container invocation passes the same `sessionId`, allowing Claude Code to resume the conversation from its `.jsonl` file in the per-group `.claude/` directory (bind-mounted at `/home/node/.claude`). Conversation history survives container teardown.

Current limitation: **one session per group folder**. All conversations in a group share one Claude Code session. See the threading document for how this is being addressed upstream.

---

## Container Lifecycle

Each invocation spawns a fresh `--rm` Docker container. Mounts per invocation:

| Container path | Host path | Access |
|---|---|---|
| `/workspace/group` | `groups/{folder}/` | read-write |
| `/workspace/global` | `groups/global/` | read-only (non-main) |
| `/workspace/project` | Project root | read-only (main only) |
| `/home/node/.claude` | `data/sessions/{folder}/.claude/` | read-write |
| `/workspace/ipc` | `data/ipc/{folder}/` | read-write |
| `/app/src` | `data/sessions/{folder}/agent-runner-src/` | read-write |

The per-group `.claude/` directory is isolated — groups cannot access each other's session history.

`CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` is set in every container's `settings.json`, meaning Claude Code auto-loads `CLAUDE.md` from **all mounted directories**, not just the working directory. This is the mechanism used for shared personality files.

Source: `src/container-runner.ts`

---

## Multiple Personalities

NanoClaw is not limited to a single agent identity. Each group has its own `CLAUDE.md` which defines the agent's name, personality, and behavior. Groups are fully isolated — memory, sessions, schedules, everything.

### Example: Victor (a separate agent identity)

Victor is a flying club assistant for `flying@cpsa.aero`, distinct from Papa (the admin agent).

**Folder structure:**

```
groups/
  victor/               # base identity — shared, never instantiated as a real group
    CLAUDE.md           # Victor's personality, role, knowledge of CPSA
  victor-shared/        # cross-channel memory — read-write for all Victor groups
    CLAUDE.md           # index of what's stored here
    members.md          # CPSA members Victor knows about
  telegram_victor/      # Victor's Telegram presence
    CLAUDE.md           # thin: Telegram formatting rules
  email_flying-cpsa/    # Victor's email presence
    CLAUDE.md           # thin: email formatting, threading behavior
```

**Mount layout for each Victor container:**

| Container path | Host path | Access |
|---|---|---|
| `/workspace/group` | `groups/telegram_victor/` or `groups/email_flying-cpsa/` | read-write |
| `/workspace/victor-base` | `groups/victor/` | read-only |
| `/workspace/victor-shared` | `groups/victor-shared/` | read-write |
| `/workspace/global` | `groups/global/` | read-only |

Because `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` is set, Claude Code auto-loads `CLAUDE.md` from all mounted paths. Victor's core personality loads from `/workspace/victor-base/CLAUDE.md` automatically — no code changes required.

`groups/victor/` is pure data (a single `CLAUDE.md`); it has no JID, no DB row, and is never instantiated as a runnable group.

`victor-shared/` gives Victor cross-channel memory — something learned in a Telegram conversation is available when responding to an email thread. This uses the existing `containerConfig.additionalMounts` mechanism. No code changes needed.

---

## Key Source Files

| File | Purpose |
|---|---|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel self-registration |
| `src/router.ts` | Message formatting, outbound routing |
| `src/container-runner.ts` | Container spawn, mount configuration |
| `src/db.ts` | All SQLite operations |
| `src/types.ts` | Shared type definitions |
| `groups/{name}/CLAUDE.md` | Per-group agent identity and instructions |
