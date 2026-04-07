# dispatch

## What it is

dispatch is a lightweight open-source CLI that lets you move context between Claude Code and Codex CLI without losing your place. Two commands, both user-triggered, both tmux-aware.

## Commands

### `dispatch fork [claude|codex] [--new-pane|--replace]`

Snapshots your current session history and opens the target tool with the full conversation as context. Use when you want to continue the same work in a different tool — e.g. you've hit Claude's rate limit, or you want Codex's opinion on the same problem.

### `dispatch handoff [claude|codex] "<brief>" [--new-pane|--replace]`

Opens the target tool in a fresh session with a brief the LLM generated. The LLM writes the brief and calls this command directly — you trigger the LLM to do so by typing something like `/handoff codex` in your session. dispatch receives the brief as an inline argument and handles the rest.

The LLM integration ships as a `dispatch init` command that writes the tool definition into your `CLAUDE.md` or Codex system prompt, so both tools know when and how to call `dispatch handoff`.

## Behaviour

- Both commands open in a new tmux pane by default
- `--replace` closes the current pane and takes it over
- You are always in control — you trigger the LLM to generate the handoff, the LLM triggers dispatch
- Works from inside any tmux session, errors gracefully if not in tmux

## Non-goals

- No automatic unprompted LLM-triggered dispatching
- No syncing or two-way session state
- No support for tools other than Claude Code and Codex CLI (for now)

## Stack

- Single Node.js script, no framework
- Reads Claude Code `.jsonl` session files and Codex history JSON for `fork`
- Shells out to tmux for pane management
- Zero runtime dependencies (ships as a standalone script)

## OSS plan

- MIT licence
- Repo: `github.com/[you]/dispatch`
- Ships as an npm package: `npm install -g @[you]/dispatch`
- README covers install, both commands, tmux requirements, session file locations, and the `dispatch init` setup step
