# dispatch

Move context between Claude Code and Codex CLI without losing your place.

## Install

```bash
ln -s "$(pwd)/bin/dispatch" ~/.local/bin/dispatch
dispatch init
```

`dispatch init` installs the `handoff` skill at `~/.agents/skills/handoff/SKILL.md`. Your `~/.agents/skills/` directory must already exist (it's your shared skills convention, not dispatch's to create).

## Commands

### `dispatch fork <claude|codex> [--new-pane|--replace] [--source <claude|codex>] [--from <id>]`

Continue the current session in another tool. Resolves the source session via most-recently-modified `.jsonl` in the current cwd across both tools' session dirs.

- **Same tool** (e.g. `dispatch fork claude` from inside Claude): delegates to native fork (`claude --resume <id> --fork-session` / `codex fork <id>`).
- **Cross tool**: snapshots the source `.jsonl` to `~/.dispatch/snapshots/`, launches the target with a tiny pointer prompt that names the snapshot path.

### `dispatch handoff <claude|codex> "<brief>"|- [--new-pane|--replace]`

Open the target tool in a fresh session with a curated brief as the first prompt. Pass `-` to read the brief from stdin (heredoc-friendly). Invoked by an LLM via the installed `handoff` skill — you trigger it by saying "handoff to claude/codex" in your session.

### `dispatch init`

Install the `handoff` skill. Re-run after a `dispatch` upgrade to refresh the skill template.

## Pane behaviour

Both `fork` and `handoff` open in a new tmux pane by default (`tmux split-window -h`). Use `--replace` to replace the current pane (`tmux respawn-pane -k`). Hard-fails outside tmux.

## File layout

```
~/.local/bin/dispatch                          # symlink to bin/dispatch
~/.dispatch/snapshots/<ts>-<uuid>.jsonl        # cross-tool fork snapshots (manual cleanup)
~/.agents/skills/handoff/SKILL.md              # the handoff skill (installed by `dispatch init`)
```
