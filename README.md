# skills

Agent skill for [Inertia.js](https://inertiajs.com) v3 on [Goravel](https://goravel.dev), built on the official [`github.com/goravel/inertia`](https://github.com/goravel/inertia) adapter.

[![skills.sh](https://skills.sh/b/cceglia/skills)](https://skills.sh/cceglia/skills)

## Install

```bash
npx skills add cceglia/skills
```

The CLI installs the skill into the current project. Add `-g` to install it for every project:

```bash
npx skills add cceglia/skills --skill goravel-inertia -a opencode -y
```

## The skill

| Skill | Use it for | References |
| --- | --- | --- |
| [`goravel-inertia`](skills/goravel-inertia/SKILL.md) | Adapter setup, web middleware and shared props, rendering, deferred/optional/once/merge/scroll props, validation, flash, redirects, asset versioning, Vite, SSR — plus the Inertia protocol and client semantics behind them: page objects, `X-Inertia` headers, partial reloads, Precognition, adapter conformance. | [adapter v1.18.0](skills/goravel-inertia/versions/v1.18.0.md), [protocol v3](skills/goravel-inertia/versions/inertia-v3.md) |

The protocol reference keeps the framework-independent contract; the adapter reference keeps the Goravel behavior, including where it diverges. The adapter wins for a project: `goravel-inertia` v1.18.0 exposes flash as `props.flash` (not the protocol's top-level field) and does not implement Precognition.

Each `SKILL.md` reads the installed version first (`go.mod` for the adapter, the client package for Inertia.js) and picks the matching reference. Reference files stay out of the initial skill load and are read only when they apply.

## Layout

```text
skills/
└── goravel-inertia/
    ├── SKILL.md
    └── versions/
        ├── v1.18.0.md
        └── inertia-v3.md
```

The skill directory name matches the `name` in its frontmatter, as the [Agent Skills specification](https://agentskills.io/specification) requires.
