# dispatch — v1 spec

Local-only first version. Single-user. No OSS scaffolding yet.

## What it is

A CLI that moves context between Claude Code and Codex CLI without losing your place. Three commands:

- **`dispatch fork`** — user-facing CLI. Snapshots the current session (or delegates to native fork for same-tool) and opens the chosen tool in a new tmux pane (or replaces the current one).
- **`dispatch handoff`** — invoked by an LLM via a skill. The LLM writes a curated brief; dispatch launches the other tool with the brief as the first prompt.
- **`dispatch init`** — one-time setup. Installs the handoff skill at `~/.agents/skills/handoff/SKILL.md`. Nothing else.

The LLM never sees `dispatch fork`. The skill never references dispatch internals beyond the literal `dispatch handoff` invocation it shells out.

## `dispatch fork`

### Invocation

```
dispatch fork <claude|codex> [--new-pane|--replace] [--source <claude|codex>] [--from <session-id>]
```

Run from a terminal, or from inside a session via Bash-mode (since you intend to invoke it from inside a session, MRM resolution is unambiguous — the act of running dispatch updates the active session's `.jsonl`, making it the most recently modified file by definition).

### Mechanism

1. **Resolve source session.** Scan both tools' session dirs filtered to the current cwd, pick the most recently modified `.jsonl` overall. The winner determines both the source tool and the source session id.
   - Claude candidates: `~/.claude/projects/<encoded-cwd>/*.jsonl` (cwd encoding replaces `/` with `-`). Session id is the filename UUID.
   - Codex candidates: `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`, filtered by reading each file's `session_meta.payload.cwd` against the current cwd. Scan today's date partition only by default. Session id is `session_meta.payload.id`.
   - `--source <claude|codex>` forces the source tool when MRM is ambiguous (plain shell with stale sessions for both tools in the same cwd).
   - `--from <session-id>` overrides the specific source session within the chosen tool.
2. **Branch on same-tool vs cross-tool.** Compare resolved source tool to the requested target.
3. **Build the launch command.**
   - **Same-tool fork** → delegate to the tool's native fork machinery, no snapshot needed:
     - Claude: `claude --resume <source-uuid> --fork-session`
     - Codex: `codex fork <source-uuid>`
     - The native machinery handles continuity; dispatch's only job is the pane decision.
   - **Cross-tool fork** → snapshot + pointer:
     - Copy the resolved `.jsonl` to `~/.dispatch/snapshots/<unix-ts>-<source-uuid>.jsonl`. Immutable from this point — fork sees the source frozen at fork time, immune to later mutation.
     - Build a tiny first prompt:
       ```
       This session is a fork of <source-tool>. The full prior session
       is at <snapshot-path>. Read it for context as needed.
       ```
     - Launch: `<target> "<prompt>"`.
4. **Place the launch command in tmux.**
   - `--new-pane` (default): `tmux split-window -h '<launch-cmd>'`
   - `--replace`: `tmux respawn-pane -k '<launch-cmd>'`
5. **Hard-fail outside tmux.** If `$TMUX` is unset, exit with: `dispatch requires an active tmux session.`

### Why this shape

- Both `claude` and `codex` accept a starting prompt as a positional argument, so dispatch never has to forge native session files or translate schemas across tools. The receiving tool starts a clean, native session and reads the snapshot lazily on its own — only the bytes it actually needs.
- "Render the transcript" was rejected: the receiving LLM is perfectly capable of parsing JSONL envelopes from the other tool's schema. Stripping or reformatting just wastes dispatch code and loses fidelity.
- Snapshot (vs live source path) prevents the surprise where a still-open source session keeps growing and the fork sees "future" turns on re-read.

### Same-tool fork

`dispatch fork claude` from inside Claude (or `dispatch fork codex` from Codex) is supported and delegates to the tool's native fork machinery (`claude --resume <id> --fork-session` / `codex fork <id>`). dispatch's value-add in this case is purely the pane decision: same-pane (`--replace`) vs new-pane (`--new-pane`, default). No snapshot is taken — the native machinery handles continuity end-to-end.

This means `dispatch fork <tool>` works uniformly whether the source and target are the same or different. The branching happens internally based on what MRM resolves the source to be.

## `dispatch handoff`

### Invocation (CLI surface)

```
dispatch handoff <claude|codex> "<brief>" [--new-pane|--replace]
```

Same tmux logic as fork. The brief becomes the literal first prompt — no snapshot, no source attached. Fresh, intentional, curated start.

### How the LLM gets here

A skill at `~/.agents/skills/handoff/SKILL.md` (single file, both tools see it — Codex reads `~/.agents/skills/` natively; Claude has `~/.claude/skills/` symlinked to it). The skill teaches the LLM:

1. **When to invoke.** When the user says "handoff to claude/codex" or invokes `/handoff <target>`.
2. **How to write the brief.** Soft template with named sections (skip any that don't apply):
   - **Goal** — what we're trying to achieve.
   - **State** — where we are now: decisions made, what's working, what's blocked.
   - **Files** — paths the receiver should read.
   - **Ask** — exactly what you want the receiver to do in this fresh session.
3. **How to ship it.** `dispatch handoff <target> "<brief>"`. Do not ask the user to confirm — write the brief and ship it.

The skill is the only place the LLM is taught about dispatch. It never sees `dispatch fork`.

### Why a skill (not a slash command + AGENTS.md block)

- One file, one location, both tools read it. No asymmetric trigger UX, no per-tool surgery, no `dispatch init` to write multiple files into multiple homes.
- Skills are the right primitive for "LLM, here's a capability" — slash commands are a thinner layer that maps to the same idea but with worse cross-tool support.

## Setup (one-time)

1. Put `dispatch` on `PATH`: e.g. `~/.local/bin/dispatch` (single Node script, no deps).
2. Run `dispatch init` to install the handoff skill at `~/.agents/skills/handoff/SKILL.md`. Confirm `~/.claude/skills/` resolves to `~/.agents/skills/` via the existing symlink.
3. Done.

### `dispatch init`

Tightly scoped: install the handoff skill, nothing else.

1. Verify `~/.agents/skills/` exists (error helpfully if not — that's your shared skills convention, not dispatch's to create).
2. `mkdir -p ~/.agents/skills/handoff/`.
3. Write `SKILL.md` (overwrite if present, so re-running after a `dispatch` upgrade pulls in skill template improvements). Local edits to the skill are clobbered on re-run — acceptable for v1.
4. Print: `Installed handoff skill → ~/.agents/skills/handoff/SKILL.md`.

No PATH setup, no symlink management, no `~/.dispatch/snapshots/` creation (`dispatch fork` mkdir-p's that lazily on first use), no `--uninstall` flag (manual `rm -rf` is fine). The skill content lives in `plan/skill.md`.

## File layout

```
~/.local/bin/dispatch                          # the script
~/.dispatch/snapshots/<ts>-<uuid>.jsonl        # fork snapshots (never auto-cleaned in v1)
~/.agents/skills/handoff/SKILL.md              # the handoff skill
```

## Decisions log

| Decision | Choice | Rationale |
|---|---|---|
| Fidelity | Full session, lazy-read | Receiver parses JSONL fine; rendering loses info and adds dispatch code |
| Delivery (fork) | Snapshot + pointer prompt | Bounded immutable artifact; tiny first turn; no quoting/argv hell |
| Delivery (handoff) | Brief as literal positional `[prompt]` arg | Both CLIs accept it natively; no file forging |
| Source resolution | MRM across both tools' dirs in cwd, `--source` / `--from` overrides | Bulletproof when invoked from inside a session (live mtime); detects source tool without env-var sniffing |
| Same-tool fork | Delegate to native (`claude --resume … --fork-session` / `codex fork`) | Reuses native continuity; dispatch only owns the pane decision |
| Tmux outside tmux | Hard-fail | Predictable; no surprise process spawning |
| LLM trigger | Skill at `~/.agents/skills/handoff/` | Single install location, both tools read it |
| Brief shape | Soft template (Goal/State/Files/Ask) | Consistent enough to skim, flexible enough to capture nuance |
| `dispatch init` | Tightly scoped: install the handoff skill only | One reproducible setup step instead of manual file drops; ~30 lines of code |
| Brief delivery to `dispatch handoff` | Heredoc from stdin (`- <<'BRIEF'`) | Avoids shell-escaping pitfalls for multi-paragraph briefs; LLMs handle heredocs reliably |

## Open minors (recommendations, not blockers)

- **Naming clash with native `--fork-session` / `codex fork`.** Keep `dispatch fork` for v1 — your muscle memory is already calibrated to the brief. The same-tool path now wraps the native fork rather than competing with it, which softens the clash. Revisit before any OSS push (candidates: `dispatch carry`, `dispatch port`, `dispatch jump`). Cheap to rename later.
- **Snapshot cleanup.** No auto-cleanup in v1. `~/.dispatch/snapshots/` is yours to nuke when it grows. Add a `dispatch snapshots prune --older-than Nd` later if it becomes annoying.

## Out of scope for v1

- Auto-cleanup of snapshots
- Tools other than Claude Code and Codex CLI
- Two-way session sync or live mirroring
- Renaming around the native fork-verb collision (defer to OSS time)
- Privacy / secret-stripping in snapshots (acceptable for local-only use)
