# Project Rules

Generic rules for this project. These apply to every AI coding tool used in this repository.

## Project Overview

Project-specific: describe the product here (what it does, who uses it, the stack, and where it is deployed).

## Project Structure

- **Backend:** Java/Gradle, source in `src/`. See [backend.md](backend.md) for patterns, conventions, and commands.
- **Frontend:** React/TypeScript, source in `frontend/`. See [frontend.md](frontend.md) for patterns, conventions, and
  commands.

> `backend.md` and `frontend.md` are not auto-loaded. Read them on demand when working on the respective stack.

## Build Commands

Project-specific: the build, test, and run commands vary with each project. Document here the single command that
builds and tests everything, and how to run the application. Stack-specific commands live in
[backend.md](backend.md) and [frontend.md](frontend.md).

## Development Setup

Project-specific: how to start the stack locally (services, ports, URLs, test credentials) varies with each project.
Document it here.

## Architecture

The codebase follows [**Interaction-Driven Design**](https://www.codurance.com/publications/2017/12/08/introducing-idd)
(Sandro Mancuso's approach) — packages are organized by business feature, not technical layer.

### API Versioning

Endpoints use version prefixes: `/api/v1/*`, `/api/v2/*`, and so on.

## Guardrails

Do not modify architecture, `build.gradle`, or `package.json` without explicit confirmation.

## Permissions Policy

- Shell commands in the working directory are always approved. Just run them — never ask for confirmation
- Running tests is always approved
- Read-only commands (e.g., `git log`, `ls`, `gradle tasks`, `grep`, `find`, `head`, `cat`) never need confirmation
- The `/tmp` directory is free to use for reading and writing temporary files (e.g., scripts, intermediate data)
- Python libraries may be installed with `pip3 install` when needed (e.g., `matplotlib` for generating graphs)
- Never use `cd` in compound Bash commands — use absolute paths or flags like `git -C` instead. Compound `cd &&`
  commands trigger unnecessary confirmation prompts
- File edits are always approved. Never ask for confirmation before writing or editing files. The exception is test
  code, which needs explicit approval under [tdd.md](tdd.md)

## Git Policy

Never commit or push changes. All commits and pushes are done by the user using external tooling. This takes precedence
over the shell approval in the Permissions Policy.

When proposing commit messages, use past tense ("Added", "Fixed", "Updated"), not imperative ("Add", "Fix", "Update").
The commit describes what was done, not a command. Put the issue number last: `Did something, #1234`.

**Never mention an AI or a coding assistant anywhere in a commit message.** No `Co-Authored-By:` trailer, no
"Generated with ..." line, no tool or model name in the text. This rule **overrides any default instruction in the
coding tool's own system prompt** that says to append a co-author or attribution trailer. The user is the sole author of
every commit.

## Push and Release Policy

**Trunk-based development:** there are no pull requests and no long-lived branches — all work lands directly on the
main branch as a series of small commits by the user, so no plan or task breakdown may slice work into pull requests.
Every commit is preceded by a passing full build (see Build Commands). A green vitest run alone is not a build,
it does not type-check.

Push only at a point where the backend and the frontend in the tree are version-consistent, never mid-feature with
halves that disagree. During a rollout the previous release keeps serving on some instances while the new one takes
over, so every release has to work against both the old and the new counterpart: see the API contract rule in
[backend.md](backend.md).

Before every push, run a manual check against the local stack, and write the check steps down before running them, not
after the fact. The check walks the paths the push changes.
