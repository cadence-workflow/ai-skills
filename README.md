# Cadence AI Skills (TEST TEST TEST)

A library of Agent Skills for working with [Cadence](https://cadenceworkflow.io/) — the fault-tolerant, stateful workflow orchestration platform.

> **Status:** Early development. The first skill targets the Go, Java, and Python SDKs with the same six-file topic layout per SDK. Coverage depth still follows the upstream SDK surfaces; the Python SDK itself is alpha so the Python files call out gaps explicitly. Additional skills are on the roadmap.

## What is this repo?

This repository is a collection of Cadence-focused **Agent Skills** — structured bundles of instructions, references, and worked examples that teach coding agents how to build, debug, and operate Cadence applications correctly.

Every skill in `skills/` is a self-contained folder with a `SKILL.md` entry point (markdown with YAML frontmatter) and a `knowledge/` folder of supporting material. Any coding agent that can load skills from a directory can use them.

## Available skills

| Skill | Description | SDK coverage |
| --- | --- | --- |
| [`cadence-developer`](./skills/cadence-developer) | Build, debug, and manage Cadence workflows, activities, and workers. | Go, Java, Python (Python SDK is alpha) |

### Roadmap

- Track upstream Python SDK changes as `cadence-python-client` matures past alpha (replay-aware logger, queries, child workflows, side effects, versioning)
- `cadence-operator` — cluster operations, configuration, archival, cross-DC replication
- `cadence-migration` — guidance for upgrading older Cadence deployments
- A shared canonical glossary across SDKs

## Using a skill

From a checkout of this repository, the simplest path is the [`skills` CLI](https://whatisskills.com), which installs into the agent-specific skills directory it detects on your system:

```bash
npx skills add .
```

To install only the `cadence-developer` skill instead of every skill in the repo:

```bash
npx skills add . --skill cadence-developer
```

The CLI auto-detects supported coding agents (Cursor, Claude Code, Codex, and others — see the [supported agents list](https://github.com/vercel-labs/add-skill#supported-agents)). Add `-a <agent>` to scope the install to a specific one, or `-g` to install globally for the current user instead of the current project. The full option reference lives at <https://github.com/vercel-labs/add-skill>.

If you'd rather install by hand, copy the skill folder into wherever your coding agent loads skills from. For Cursor's global skills directory:

```bash
mkdir -p ~/.cursor/skills
cp -R skills/cadence-developer ~/.cursor/skills/
```

Consult your agent's documentation for the exact skills directory it watches.

## Repository layout

```
ai-skills/
├── skills/                       # individual Agent Skills, one folder each
│   └── cadence-developer/
│       ├── SKILL.md              # entry point with YAML frontmatter
│       └── knowledge/            # supporting reference material
│           ├── shared/           # language-agnostic Cadence concepts
│           ├── go/               # Go SDK specifics
│           ├── java/             # Java SDK specifics
│           └── python/           # Python SDK specifics (alpha)
├── AGENTS.md                     # rules for agents working IN this repo
├── CONTRIBUTING.md               # how to add or update a skill
├── LICENSE                       # Apache 2.0
└── README.md
```

## Sources of truth

Skill content is grounded in the upstream Cadence projects:

- [`cadence-workflow/cadence`](https://github.com/cadence-workflow/cadence) — server
- [`cadence-workflow/cadence-go-client`](https://github.com/cadence-workflow/cadence-go-client) — Go SDK
- [`cadence-workflow/cadence-java-client`](https://github.com/cadence-workflow/cadence-java-client) — Java SDK
- [`cadence-workflow/cadence-python-client`](https://github.com/cadence-workflow/cadence-python-client) — Python SDK
- [`cadence-workflow/Cadence-Docs`](https://github.com/cadence-workflow/Cadence-Docs) — official documentation

When skill guidance and the upstream source disagree, the upstream source wins. Please open an issue or PR.

## Contributing

Contributions are welcome. See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for how to add a new skill, evolve an existing one, or improve the knowledge base.

## License

Apache 2.0 — see [`LICENSE`](./LICENSE).
