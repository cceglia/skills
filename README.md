# skills

Agent skills for [Inertia.js](https://inertiajs.com) and its [Goravel](https://goravel.dev) adapter.

[![skills.sh](https://skills.sh/b/cceglia/skills)](https://skills.sh/cceglia/skills)

## Install

```bash
npx skills add cceglia/skills
```

The CLI lists the skills in this repository and installs the ones you choose into the current project. Add `-g` to install them for every project, or select explicitly:

```bash
# Both skills, non-interactive
npx skills add cceglia/skills --skill inertia-core --skill goravel-inertia -a opencode -y

# List what this repository offers
npx skills add cceglia/skills --list
```

## Skills

| Skill | Use it for | Versions |
| --- | --- | --- |
| [`inertia-core`](skills/inertia-core/SKILL.md) | Inertia.js protocol and client semantics, independent of server and UI frameworks: page object, headers, prop categories, redirects, adapter conformance. | [v3](skills/inertia-core/versions/v3.md) |
| [`goravel-inertia`](skills/goravel-inertia/SKILL.md) | The `github.com/goravel/inertia` adapter: installation, web middleware, rendering and props, validation, flash, redirects, asset versioning, Vite and SSR. | [v1.18.0](skills/goravel-inertia/versions/v1.18.0.md) |

They are designed to be used together: `goravel-inertia` covers the adapter, `inertia-core` covers the protocol it implements. The adapter skill works on its own, but protocol details are deferred to the core skill.

Each `SKILL.md` reads the installed version first (`go.mod` for the adapter, the client package for Inertia.js) and picks the matching reference under `versions/`. Reference files stay out of the initial skill load and are read only when the version applies.

## Layout

```text
skills/
├── goravel-inertia/
│   ├── SKILL.md
│   └── versions/
│       └── v1.18.0.md
└── inertia-core/
    ├── SKILL.md
    └── versions/
        └── v3.md
```

Each skill is a directory whose name matches the `name` in its frontmatter, as the [Agent Skills specification](https://agentskills.io/specification) requires.
