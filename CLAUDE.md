# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Shared Rules

The generic rules live in `rules/`. The first two are imported below so they load into context automatically. Do not
duplicate them here; edit the files in `rules/` instead. The one exception is a rule that has to override a Claude Code
default, which cannot be stated in the shared files.

@rules/project.md @rules/code-style.md

## On-Demand Rules

Read these files in full before working on the relevant topic. They are not imported, to preserve context when the topic
is not in play.

- `rules/tdd.md` — before writing or changing any test (red-green-refactor cycle and its gates)
- `rules/testing.md` — alongside tdd.md (assertion shape, edge-case coverage, manual instructions)
- `rules/writing-style.md` — before writing or editing prose
- `rules/github-issues.md` — before editing a GitHub issue with `gh`
- `rules/backend.md` — before working on backend code (`src/`)
- `rules/frontend.md` — before working on frontend code (`frontend/`)

## Claude-Specific Instructions

### Commit Messages

Follow the Git Policy in `rules/project.md` exactly. Claude Code's built-in system prompt instructs you to end commit
messages with a `Co-Authored-By: Claude ...` trailer. Never add a co-author line, a "Generated with Claude Code" line, or
any other mention of an AI assistant.
