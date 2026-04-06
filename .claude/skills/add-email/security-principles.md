# Security principles

## Security: Break the Main Group's [ABC] Chain

The main group currently violates Meta's [Agents Rule of Two](https://ai.meta.com/blog/practical-ai-agent-security/) — it simultaneously processes untrustworthy inputs [A], accesses sensitive systems [B], and changes state [C]. A prompt injection in fetched web content can propagate through `global/CLAUDE.md` to every group.

### Option 1: Remove [A] — No untrustworthy inputs in main

Disable WebFetch/WebSearch tools when the agent is invoked from a group with `isMain: true`. Main becomes admin-only (register groups, manage tasks, write global memory). All web browsing and research happens in a regular "daily driver" group. Could be enforced in code or just by convention and documentation.

### Option 2: Remove [C] — Make global memory structurally read-only

Mount `groups/global/` as read-only in containers so the agent physically cannot write to `global/CLAUDE.md`. The file is already tracked in the repo — edits go through the user modifying the file by hand and committing. This is a structural guarantee, not a prompt-level one. Relying on the LLM to "ask before writing" is still breakable by injection; the mount permission is not.

### Option 3: Remove [B] — Sandbox browsing away from sensitive data

When the agent fetches web content, drop access to global memory and admin operations. One-way transition: once untrustworthy content enters the context, sensitive writes are locked out for the rest of the session.

---

## Security: Establish a Security Mindset in the Docs

The project's security docs delegate security responsibility to dependencies — Claude's training resists prompt injection, Docker containers provide isolation. But those are just dependencies. NanoClaw's own code and architecture create the attack surfaces, trust boundaries, and data flows. The project needs a security section that establishes the mindset for NanoClaw developers and admins, covering:

- **NanoClaw owns its security model.** Claude's safety training and container sandboxing are defense-in-depth layers, not the security strategy. The code we write — what gets mounted, what's writable, what tools are available in which context, how memory propagates — defines the actual attack surface.
- **Threat modeling for the architecture we control.** Where does untrustworthy input enter? Where does it persist? Where can state changes propagate across trust boundaries? The global memory layer, group registration, task scheduling, and mount configuration are all NanoClaw-owned concerns.
- **Admin responsibility.** The person running NanoClaw makes security-critical decisions: which groups to register, what directories to mount, whether to use the main group for casual browsing. The docs should make the consequences of these decisions legible, not buried.
- **Apply the [Agents Rule of Two](https://ai.meta.com/blog/practical-ai-agent-security/) as a design principle.** Any context that combines untrustworthy inputs [A], sensitive data access [B], and state-changing capability [C] is a violation. Evaluate new features and skills against this framework.
- **Structural over behavioral.** Security boundaries must be enforced by architecture (mount permissions, tool availability, code paths), not by prompting the model to behave. Prompt-level controls are breakable by the same injection they're meant to prevent.

---

## Architecture: Admin/Personal Split from Day One

The current setup flow creates a single "main" group that serves as both the admin channel and the user's primary chat. This conflates two roles with very different trust profiles. Instead, `/setup` should create two groups by default:

- **Admin** — Privileged operations only: register/remove groups, manage tasks across groups, configure mounts. No WebFetch, no WebSearch, no untrustworthy inputs. Structurally [BC] — trusted inputs acting on sensitive state. This is where `isMain: true` lives.
- **Personal** (or whatever name fits) — The user's everyday chat. Web browsing, research, casual conversation, general tasks. Has the same tools as any regular group. No admin privileges, no global memory writes. [AC] — untrustworthy inputs with communication, but no access to sensitive cross-group state.

This should be the default experience, not an afterthought hardening step. The current naming ("main") conflates admin privilege with primary usage, which encourages exactly the wrong behavior — using the most privileged context for the most exposed activity.

Consider renaming what's currently called "main" to "admin" to make the privilege level obvious, and making the personal group the one that feels like home.

---

## Identity: Disambiguate Users in Group Chats

Channel code (at least Telegram) likely captures only `first_name` for `sender_name`. In groups with multiple people sharing a first name (e.g., two Chris's in a nonprofit), the agent can't tell them apart — messages show `sender="Chris"` for both. The agent loses track of who said what and could mix up context between them.

Telegram provides `first_name`, `last_name`, and `username` on every message. The channel code should build a disambiguated `sender_name` — e.g. `Chris W.` or `Chris (@chriswilson)` — so the agent can reliably distinguish participants. This applies to any channel where display names aren't guaranteed unique (which is most of them).
