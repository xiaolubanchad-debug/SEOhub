# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This repository is newly created and currently contains no source code, build configuration, or tests. Directory name suggests an SEO-related project (`SEOhub`), but no scope, stack, or architecture has been decided yet.

When the user asks you to build something here, first clarify:
- Target stack (e.g. Node/TypeScript, Python, Next.js, etc.)
- Whether this is a web app, CLI, library, or service
- Any external APIs or data sources involved

Do not scaffold a stack or framework unprompted — wait for the user to choose.

## Behavioral Guidelines

[SKILL.md](SKILL.md) contains the **karpathy-guidelines** — behavioral rules for code changes in this repo. Follow them:

- **Think before coding** — state assumptions, surface tradeoffs, ask when unclear instead of guessing.
- **Simplicity first** — minimum code that solves the problem; no speculative abstractions, flexibility, or error handling for impossible cases.
- **Surgical changes** — touch only what the task requires; don't "improve" adjacent code, don't refactor what isn't broken, match existing style.
- **Goal-driven execution** — turn tasks into verifiable success criteria (e.g. "write a failing test, then make it pass") before looping.

These apply in addition to the standing rules in the system prompt and take precedence when more specific.
