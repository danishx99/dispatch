# dispatch

Move context between [Claude Code](https://github.com/anthropics/claude-code) and [Codex CLI](https://github.com/openai/codex) without losing your place.

You're deep into a session in one CLI and want a second opinion from the other. Or one model is stuck and you want to swap. Or you're done with a planning phase in one tool and want to execute it in the other. `dispatch` is the tiny piece of glue that makes that a single command instead of a copy-paste exercise.

```
dispatch fork codex          # continue this session in a new Codex pane
dispatch fork claude         # continue this session in a new Claude pane
/handoff codex               # let Claude write a brief and ship it to a fresh Codex session
```

## How it works

- **`dispatch fork`** finds your current session by looking up the most recently modified `.jsonl` in the right place under `~/.claude/projects/` or `~/.codex/sessions/` for your current working directory. If you fork into the *same* tool, it delegates to the tool's native fork (`claude --resume … --fork-session` or `codex fork …`) so prior turns are preserved exactly. If you fork into the *other* tool, it snapshots the source `.jsonl` to `~/.dispatch/snapshots/` and starts the target with a tiny pointer prompt — the receiving model reads the snapshot for context as needed.

- **`dispatch handoff`** is the LLM-driven path. The installed `handoff` skill teaches Claude (or Codex) to write a curated four-section brief — Goal, State, Files, Ask — and ship it via `dispatch handoff <target> - <<'BRIEF' … BRIEF`. The receiver gets a fresh session with just the brief as context, no prior history. Use this when you want a clean restart with a clear handoff, not a full session continuation.

Both commands open the new session in a fresh tmux pane to the right (`tmux split-window -h`). Pass `--replace` to replace the current pane instead.

## Requirements

- [tmux](https://github.com/tmux/tmux) — dispatch hard-fails outside tmux
- [Claude Code](https://github.com/anthropics/claude-code) and/or [Codex CLI](https://github.com/openai/codex) on your `PATH`
- Node.js ≥ 18

## Install

```bash
npm i -g dispatch-cli
dispatch init
```

`dispatch init` interactively installs the `handoff` skill into the skill directories you pick. It detects `~/.claude/skills/` and `~/.agents/skills/` and offers each as a checkbox; existing dirs are pre-ticked.

To install from source instead:

```bash
git clone https://github.com/danishx99/dispatch
ln -s "$(pwd)/dispatch/bin/dispatch" ~/.local/bin/dispatch
dispatch init
```

## Commands

### `dispatch fork <claude|codex> [--new-pane|--replace] [--source <claude|codex>] [--from <id>]`

Continue the current session in another tool.

- `<target>` — `claude` or `codex`. Same as the source tool means a same-tool fork (delegates to native fork machinery). Different tool means a cross-tool fork (snapshot + pointer prompt).
- `--new-pane` *(default)* — open the new session in a new tmux pane to the right.
- `--replace` — replace the current pane instead.
- `--source <claude|codex>` — restrict source-session resolution to one tool. Without this, dispatch picks the most-recently-modified session across both tools in your current cwd.
- `--from <id>` — pin to a specific session UUID instead of MRM. Requires `--source`.

### `dispatch handoff <claude|codex> "<brief>"|- [--new-pane|--replace]`

Open the target tool in a fresh session with a brief as the first prompt. Pass `-` as the brief to read from stdin (heredoc-friendly). Usually invoked by an LLM via the installed `handoff` skill — you trigger it by saying "handoff to claude" or "handoff to codex" in your session.

### `dispatch init`

Interactively install the `handoff` skill into your skill directories. Re-run after upgrading dispatch to refresh the skill template. Non-interactive (no TTY): installs to all detected skill directories, errors if none exist.

## File layout

```
$(npm root -g)/dispatch-cli/                   # package install
~/.local/bin/dispatch                          # CLI symlink (managed by npm)
~/.dispatch/snapshots/<ts>-<uuid>.jsonl        # cross-tool fork snapshots (manual cleanup)
~/.claude/skills/handoff/SKILL.md              # installed handoff skill
~/.agents/skills/handoff/SKILL.md              # installed handoff skill (if you use this convention)
```

## Why

I use both Claude Code and Codex CLI heavily, often in the same project. Switching between them used to mean either losing context entirely (start fresh and re-explain) or copy-pasting half a conversation. Native fork helps within a tool, but there's no native bridge between them. `dispatch` is that bridge — small enough to read in one sitting, no dependencies, no daemon, no background process.

## License

MIT
