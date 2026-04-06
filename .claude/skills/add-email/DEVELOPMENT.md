# Fork development philosophy

How this repository relates to upstream NanoClaw, West Arete’s contributions, and other repos (e.g. CPSA). These are conventions for maintainers of this fork, not rules for upstream.

## Roles of each repo

| Repo | Role |
|------|------|
| [`qwibitai/nanoclaw`](https://github.com/qwibitai/nanoclaw) | Upstream — source of truth for core. Add as remote `upstream`. |
| This fork ([`westarete/nanoclaw`](https://github.com/westarete/nanoclaw)) | West Arete — public fork where we develop skills (e.g. `/add-email`), `.claude/skills/add-email/`, and open PRs upstream. `origin` points here. |
| [`westarete/nanoclaw-skills`](https://github.com/westarete/nanoclaw-skills) (when created) | West Arete marketplace — Claude Code plugin bundling feature-skill `SKILL.md` files (e.g. `/add-email`). See [PLAN.md — Community marketplace](PLAN.md#community-marketplace-westaretenanoclaw-skills). |
| `cpsa-aero/nanoclaw-cpsa` (private) | CPSA deployment — Victor, club-specific `groups/`, production config. Not part of this fork; do not mix confidential CPSA content into `westarete/nanoclaw`. |

Use separate clones per org when it keeps remotes and commit identity obvious; avoid juggling multiple unrelated `origin` remotes in one directory.

## Branches

- `main` — Tracks `upstream/main`. Prefer it fast-forwardable to upstream: merge `upstream/main` into `main` regularly. Avoid landing feature code or **this skill’s** large doc/code drops on `main` if you want `main` to stay a clean sync line. The root README on `main` has a short fork banner at the top and links [`.claude/skills/add-email/README.md` on `skill/email`](https://github.com/westarete/nanoclaw/blob/skill/email/.claude/skills/add-email/README.md); merge upstream README changes into the body below the banner as usual.

- `skill/<name>` (e.g. `skill/email`) — Feature work following NanoClaw’s [skills-as-branches](https://github.com/qwibitai/nanoclaw/blob/main/docs/skills-as-branches.md) model. Channel implementation and **`.claude/skills/add-email/`** for planning/docs live here. One capability per branch / PR to upstream when contributing back.

CPSA / Victor material belongs in the CPSA repo, not under this fork’s history.

## Staying up to date

1. `git fetch upstream`
2. `git checkout main` → `git merge upstream/main` (bring `main` current)
3. `git checkout skill/email` → `git merge main` (or `git merge upstream/main`) to refresh the skill branch

Merge is the default: it preserves history and is safe once a branch is pushed or shared. Rebase onto latest `main` is fine for a solo branch before a PR if you want a linear story; avoid rebasing branches other people have based work on.

Upstream does not care about merge commits inside your fork; they care about the PR branch you submit (clear scope, tests, [CONTRIBUTING](https://github.com/qwibitai/nanoclaw/blob/main/CONTRIBUTING.md)). Many projects squash-merge PRs anyway.

## Pull requests to upstream

- One logical change per PR.
- Keep the PR branch reasonably current with `upstream/main` (merge or rebase as appropriate before review).
- Your fork’s `main` graph is yours; reviewers focus on the PR diff, not whether your personal `main` is linear.

## `.claude/skills/add-email/`

Everything here is fork-local skill material — not part of `qwibitai/nanoclaw` unless you contribute it deliberately. `PLAN.md`, `security-principles.md`, and this file track how we work and what we’re building on `skill/email`. Commit them on the same branch as the related skill work unless you explicitly choose otherwise.
