# Red Green Leash

**Battle-tested rules that keep AI coding assistants on a short leash: test first, red, green, one small step at a time, with a human in the loop.**

> No green without red. No step without a human.

AI coding assistants are fast, eager and confident. Left alone they write a lot of code, skip the tests, and present the result as done.
These rules slow them down in the right places. The assistant must prove a test fails before it writes code, take one small step at a time, and stop to let a human decide what happens next.

This is a snapshot of the rules I use every day while developing a product test first. They are not theory: they are what remained after many sessions where the assistant ran ahead and I had to pull it back.

## The three principles

### 1. Test first – red before green
No production code without a failing test that demands it.
The assistant writes one test, runs it, and shows that it fails for the expected reason. Only then does it write the code needed to make it pass.

### 2. Small steps
One test, one behaviour, one change at a time.
Small steps keep the diff reviewable, the feedback loop short, and the way back cheap when something goes wrong.

### 3. Human in the loop
The assistant proposes, the human decides.
It stops at agreed checkpoints, such as after red, after green and before refactoring or committing, and waits for approval instead of carrying on.

## The loop

1. 🔴 **Red** – write one failing test, run it, show the failure
2. ✋ **Check in** – the human confirms that the test describes the right behaviour
3. 🟢 **Green** – write the simplest code that makes the test pass, run all tests
4. ✋ **Check in** – the human reviews the change
5. 🔵 **Refactor** – clean up with all tests green
6. 🔁 **Repeat** – pick the next small step

## What's in here

<!-- TODO: adjust to the actual files -->

| File | Purpose |
|---|---|
| `CLAUDE.md` | Rules for Claude Code |
| `AGENTS.md` | Rules for Mistral Vibe and other agents that read `AGENTS.md` |
| `rules/` | The individual rules, one file per topic |

## Getting started

<!-- TODO: adjust to the actual files -->

1. Copy the rule files into the root of your project.
2. Adapt them to your stack: test command, language, conventions.
3. Start your assistant and ask it for the next small step.

The rules are a starting point, not a standard. Remove what doesn't fit, sharpen what does, and add the rules your own project needs.

## Tested with

- [Claude Code](https://claude.com/claude-code)
- [Mistral Vibe](https://mistral.ai/)

They should work with any assistant that reads instruction files, but these are the two I use daily.

## Why "leash"?

A leash is not a cage. The assistant still does the walking and covers a lot of ground, but it stays close, and you decide the direction.

## Contributing

This is a snapshot of a personal rule set, so it reflects one way of working. Issues and pull requests that sharpen a rule, or describe where a rule failed for you, are welcome.

## License

[MIT](LICENSE) © Thomas Sundberg
