# cadence-tutor

An interactive daily curriculum that teaches you Cadence internals from the real source code. Your coding agent acts as the instructor: each day is a 10-15 minute unit — a lesson grounded in the upstream repositories, a short quiz, and a small Go exercise you implement yourself and get graded on immediately.

This is the inverse of [`cadence-developer`](../cadence-developer): that skill teaches agents to build Cadence applications; this one uses the agent to teach a person how Cadence itself works.

> **AI-generated content — review everything.**
> The lessons, quizzes, exercises, and curated track files in this skill are AI generated. The skill validates its own output (it grep-verifies source citations and requires each exercise test to fail before and pass after the reference solution), but AI can make mistakes: citations can drift as the code evolves, explanations can be subtly wrong, and generated Go code can be misleading or unsafe to copy. Treat every lesson as a guided tour, not an authority — verify claims against the [upstream source](https://github.com/cadence-workflow/cadence), and always review generated code before you run or reuse it. If you find an error, please open an issue or PR.

## Installation

From a checkout of this repository:

```bash
npx skills add . --skill cadence-tutor
```

Add `-a <agent>` to target a specific coding agent, or install by hand by copying `skills/cadence-tutor/` into the directory your agent loads skills from. See the [repository README](../../README.md) for details.

## Requirements

- A coding agent that loads Agent Skills and can run shell commands.
- Local checkouts of the repositories a track teaches from — for most tracks [`cadence-workflow/cadence`](https://github.com/cadence-workflow/cadence), and for tracks that cover the client side [`cadence-workflow/cadence-go-client`](https://github.com/cadence-workflow/cadence-go-client). The skill asks for the paths on first use and can shallow-clone missing repositories for you.
- Go (a current stable version) for the daily exercises, and Python 3 for the progress ledger.

## Getting started

Ask your agent to start a track:

```
cadence-tutor tracks                    # list available tracks and your progress
cadence-tutor start task-processing     # begin a track (scaffolds workspace + syllabus, runs day 1)
cadence-tutor next                      # run today's lesson for your in-flight track
```

On first use the skill creates a workspace at `~/cadence-courses/` and asks where your repository checkouts live, storing the answer in `~/cadence-courses/config.json`. Each track gets its own directory with a syllabus, per-day lesson files, a progress ledger, and a standalone Go module for exercises — nothing is written inside your repository checkouts.

## The daily loop

Every day follows the same four steps, about 10-15 minutes total:

1. **Read** — the agent presents a 700-1600 word lesson citing real code as `server:path/file.go:LINE`.
2. **Answer** — 3-5 quiz questions, asked interactively and graded on the spot with explanations.
3. **Do** — you implement 5-10 lines of Go in the day's exercise stub until the provided test passes (or say "skip" to see the solution).
4. **Grade** — the agent runs the test, reads your code, and compares your approach to the reference solution.

Progress is tracked per track; `cadence-tutor next` always resumes where you left off, even across sessions.

## Available tracks

| Track | Days | Covers |
| --- | --- | --- |
| `task-processing` | 20 | Transfer and timer task processing, from the history service queues to the Go SDK poller loops |
| `shard-management` | 15 | Shard identity and routing, ringpop membership, RangeID fencing, shard movement and drain |
| `replication-multicluster` | 15 | Replication tasks, NDC version histories, conflict resolution, failover, the replication DLQ |
| `matching-internals` | 15 | Task list managers, sync match, forwarding, pollers, isolation groups, sticky task lists |
| `history-mutable-state` | 15 | Mutable state, history branches, the execution cache, decision lifecycle, query and reset |

Want a topic that has no track? `cadence-tutor custom "<topic>"` generates a new track for any Cadence subject, verified against the source the same way. If it turns out well, consider contributing it here as a curated track.

## Other commands

```
cadence-tutor status [track]    # progress for one or all started tracks
cadence-tutor reset <track>     # restart a track from day 1
```

## How lessons stay honest

Lesson drafting can be delegated to a cheaper model, but the supervising agent must validate every lesson before presenting it: at least three `file:line` citations are grep-verified against your checkouts, the exercise must compile and fail before your implementation, and the reference solution must make the test pass. The syllabus is re-verified against your local checkouts when generated, so lessons track the code you actually have. None of this replaces your own review — see the notice at the top.

## Layout

```
cadence-tutor/
├── SKILL.md                    # the curriculum engine (entry point)
├── tracks/                     # curricula as data files, one per track
├── references/
│   ├── lesson-format.md        # per-day file formats and the validation checklist
│   └── drafter-briefs.md       # brief templates for delegated drafting
└── scripts/
    └── progress.py             # standard-library-only progress ledger
```
