# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code plugin called **auk-review** that provides the `/analyze-frontend-architecture` skill — a React frontend architecture reviewer.

## Plugin Structure

- `.claude-plugin/plugin.json` — Plugin manifest (name, description, version)
- `skills/analyze-frontend-architecture/SKILL.md` — The sole skill, markdown with YAML frontmatter

No build step, no tests, no dependencies. The plugin is purely markdown-based.

## How the Skill Works

User-invocable via `/analyze-frontend-architecture <path-to-repository>`. Has `disable-model-invocation: true` so it only triggers on explicit invocation.

When invoked, it analyzes a React project's component structure, SLA violations, component APIs, composition patterns, data flow, and code duplication, then writes findings to `ARCHITECTURE_REVIEW_RESULTS.md` in the target project.

## Key Architectural Opinions (Deliberate Choices)

These are intentional positions baked into the skill — not defaults to override:

- **Layout-inside-Page**: Pages compose `<AppLayout>` themselves. Never recommend router-level `<Outlet />` wrapping.
- **API layer as a class**: Service classes with constructor injection, not bare exported functions.
- **Hooks consume services, not SDKs**: Hooks never import Firebase/axios/etc. directly — they wrap the API service layer.
- **Fix duplication with layering, not merging**: Duplicate API calls across hooks → extract shared service layer, don't merge into mega-hook.

## Git Conventions

- Never add `Co-Authored-By` lines to commit messages.
