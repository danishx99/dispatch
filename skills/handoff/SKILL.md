---
name: handoff
description: Hand off the current task to a fresh session in the other CLI (Claude Code or Codex) by writing a curated brief and shipping it via the `dispatch` command. Use when the user says "handoff to claude", "handoff to codex", "/handoff <target>", "send this to <other tool>", or any equivalent phrasing. They must mention 'via dispatch'
---

# Handoff

You are about to hand off the current task to a fresh session in the other CLI. The receiver will get **only** the brief you write — no prior conversation, no tool history, no shared memory. Your brief is the entire context they have to work from. Make it complete enough that they can start being productive on the next concrete step without asking the user "what are we working on?"

## When to invoke

The user says any of:

- "handoff to claude" or "handoff to codex"
- "/handoff claude" or "/handoff codex"
- "send this to claude/codex"
- "let codex take this from here"
- equivalent intent in any phrasing

If the user says "fork" (rather than "handoff"), that is a different command (`dispatch fork`) which the user runs themselves — do not invoke this skill.

## How to write the brief

Use these four sections. Skip any that genuinely don't apply, but bias toward including all four.

### Goal

What are we trying to achieve? One or two sentences. The big picture.

### State

Where are we right now? Be concrete:

- Decisions already made (and the why)
- What is working
- What is blocked or uncertain
- Specific error messages, command output, or symptoms if relevant

Avoid summarizing in the abstract. Name files, commands, and exact strings.

### Files

Absolute paths the receiver should read first, as a bulleted list. Order them by importance — what they should read first goes first.

### Ask

Exactly what you want the receiver to do in this fresh session. One concrete next action, not "continue from here." If there are sub-steps, list them. If there are constraints or things to avoid, say so.

## How to ship the brief

Use the heredoc form so multi-paragraph content is delivered cleanly without shell-escaping pitfalls:

```bash
dispatch handoff <target> - <<'BRIEF'
## Goal
...

## State
...

## Files
- /abs/path/one
- /abs/path/two

## Ask
...
BRIEF
```

The literal `-` after `<target>` tells dispatch to read the brief from stdin. Use single-quoted heredoc delimiter (`<<'BRIEF'`) to prevent shell interpolation of `$`, backticks, etc., in the brief content.

`<target>` is `claude` or `codex` — the tool you are handing off **to**, not the one you are in.

By default the brief opens in a new tmux pane. If the user said "swap to <tool>" or "replace this with <tool>", add `--replace`:

```bash
dispatch handoff codex --replace - <<'BRIEF'
...
BRIEF
```

Do not ask the user to confirm before shipping. Write the brief, ship it, and let them know which pane it landed in.

## Length and detail

Both Claude and Codex accept large first-prompt arguments. Don't artificially compress. A long, concrete brief is better than a short, vague one. The receiver will read what they need from it.

## What not to do

- Do not omit the **Ask** section. A brief without an explicit next action forces the receiver to guess what the user wants.
- Do not ask the user to confirm or to write the brief themselves. They invoked this skill so you would do that work.
