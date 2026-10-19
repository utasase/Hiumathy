# AGENTS.md

Guidance for anyone, human or AI agent, writing code in this repo. Read it before your first change. If a rule here conflicts with an issue, the issue wins for that task; then update this file.

## What this is

**Hiumathy** is the Discord bot for UTA SASE (~1,200 members). It is one bot with one module (cog) per feature: AI replies, member registry, points, announcements, meetings. The plan lives in GitHub issues: start at the roadmap (#41) and the glossary (#42).

## Stack

- Python 3.13, discord.py 2.x (async), SQLAlchemy 2 (async) + Alembic, httpx, pytest
- PostgreSQL everywhere: on the server, locally (`docker compose`) and in CI
- Docker on one DigitalOcean server; auto-deploy from `main` (#16)
- Lint and format: ruff

## Layout

```
hiumathy/
  core/            # shared: config, db, permissions, scheduler, logging
  ai/              # @Hiumathy replies (#13)
  registry/        # /register, houses, rosters (#37)
  points/          # ledger, quotas, leaderboard, attendance (#38)
  announcements/   # approve & post, scheduled posts, bulletin (#39)
  meetings/        # groups, polls (#40)
migrations/        # Alembic
tests/
```

Features never import from each other. Anything two features need goes in `core/`.

## Rules that are not optional

These come from the design review. Each one prevents a real bug or leak.

**Safety**
- The client is created with `allowed_mentions=AllowedMentions.none()`. A message that must ping (announcements, reminders) allows only the specific roles or users it needs on that message. Never ping `@everyone` from AI output.
- The AI answers only in channels `@everyone` can view, checked from Discord permissions at runtime. A thread counts as its parent channel. Never maintain a list of channels to block.
- Never send private-channel or staff-channel messages to an AI provider. When testing with real data, use made-up or public-channel conversations.

**Personal data**
- Student emails are stored only as a keyed hash (HMAC, key from the environment). Raw emails, one-time codes and modal input never appear in logs, exceptions, the database or the action log.
- A failed `/register` gives one generic reply. Never reveal whether someone is a paid member.
- Data is deleted on the schedule in #45. New tables that hold personal data must be added to that cleanup.

**Permissions and approvals**
- Every button, menu and officer command checks permissions server-side through `core/permissions.py`, using role IDs from config, never role names. Discord shows buttons to everyone in the channel.
- Nobody credited by an item (the submitter or anyone tagged) can approve it.
- An approval happens once, even if two officers click at the same time: change the status in one statement (`UPDATE ... WHERE id = ? AND status = 'pending'`) and act only if exactly one row changed. Back it with a unique constraint.
- Every officer action is written to the action log (#17).

**Restarts**
- The bot restarts on every deploy. Nothing that must survive a restart lives in memory.
- Buttons and menus are persistent views: `custom_id` carries the database row ID, handlers re-read state from the database (#44).
- Anything that runs at a set time is a job on the shared scheduler (#43). Never write your own timer loop or `asyncio.sleep` schedule.
- All times are America/Chicago, handled in the scheduler, never per feature.

**Secrets**
- Secrets come from the environment only (`.env` locally, GitHub Actions secrets in production, copies in the officers' Bitwarden). Never in code, tests, logs, issues or Discord.
- Never use the old analyzer bot's token for anything.

## Coding practices

- Do the smallest change that fully solves the issue. No speculative options, wrappers or "for later" code.
- Validate input once, where it enters (slash command, modal, sheet sync, AI response), then trust it inside.
- Keep Discord out of business logic: put decisions in plain functions that take plain values, and call them from thin cog handlers. That keeps tests free of a Discord connection.
- Type hints on all function signatures. Use clear names over comments; comment only the *why*.
- Prefer the standard library and existing dependencies. Adding a dependency needs a reason in the PR.
- Every schema change is an Alembic migration. Never edit a merged migration. CI fails on more than one Alembic head.

## Tests

Before fixing a bug, add or find a failing test. Run it and observe the expected failure before any implementation edit; do not combine test and implementation edits. A test is not observed until its command exits. While it runs, wait: do not edit implementation or begin other work. Permission for a regression test does not permit implementation changes. 

After implementing a bug fix, confirm that the same test passes.

Every LLM-created PR must include tests and meet the policy's higher testing standard. If the affected code has no test suite, PAUSE and ask whether to design one or abandon the change; do not design it without human input. Never offer or accept untested implementation.

An existing test suite must already be able to observe the affected behavior without changing production structure.

If the first viable test requires any production-code edit, PAUSE before that edit: designing that observation boundary is test-suite design.

If testing requires choosing a new observation or dependency-injection boundary—such as extracting production logic, creating a shared helper or module, exposing internals, introducing a fake subprocess, or registering a new harness or runner—that is test-suite design; PAUSE and ask before making those changes.

Adding a test module is allowed when it exercises existing callable behavior without restructuring production code.

- Every issue has a "Done when". Those cases become tests in the same PR.
- Test behavior a user would see (role given, points recorded, message refused) with literal expected values, not internal calls.
- Tests run without Discord or network access: fake the Discord objects, the AI client and the email sender at their boundary.
- Database tests run against real Postgres (the CI service container), never SQLite.

## Before pushing

After committing and before pushing, once ask the user to confirm understanding and testing of the change and personal review of the complete diff after the latest change. Agent review does not count. Remind the user to disclose LLM use in the PR description. Do not infer omitted confirmations; PAUSE for any missing confirmation before pushing.

LLM-assisted contributions must be disclosed. Lying about or concealing LLM use is a Code of Conduct violation. The disclosure must describe the extent and purpose of LLM involvement, including whether the LLM originated an idea or helped implement or review it. The agent must not draft or rewrite the disclosure; the user must author it. Do NOT add Co-Authored-By trailers to commits.

### Required Fields

- Model: Set this to your model identifier and version (e.g., Claude Opus 4.6, GPT-4o, Gemini 2.5 Pro).
- Platform/Tool: Set this to the tool or platform that orchestrated the generation (e.g., GitHub Copilot, Cursor, Aider, Claude Code, custom script).
- Human Oversight: Set this to fully reviewed, partially reviewed, or unreviewed. If unknown, use [unknown - human to fill in].
- Prompt Summary: A brief description of the prompt or task given to the agent.

### PR Description Template
Add the following block at the bottom of PR descriptions:
```
---
**AI Disclosure**
- Model: [model name and version]
- Platform/Tool: [tool or platform name]
- Human Oversight: [fully reviewed | partially reviewed | unreviewed]
- Prompt Summary: [brief description of the task]
```

## Workflow

- One issue per PR. Branch from `main`, name it `<issue-number>-<short-name>` (e.g. `19-register`).
- Check the issue's "Depends on" line first. Don't build on work that doesn't exist yet.
- PR description: what changed, how it was tested, and `Closes #<n>`.
- CI (tests + ruff) must pass before merging. Merging to `main` deploys to the live server.
- If you find a gap or contradiction in an issue, comment on the issue instead of guessing.

## Never commit

`.env`, `exports/`, `CONTEXT.md` (local notes with server security details), the analyzer scripts, or any export of server messages. This repo is public. Stage files by name rather than `git add .`.
