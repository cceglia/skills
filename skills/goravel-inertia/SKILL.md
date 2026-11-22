---
name: goravel-inertia
description: "Build, configure, test or debug Inertia.js v3 apps on Goravel with github.com/goravel/inertia: installation, web middleware and shared props, facade rendering, deferred/optional/once/merge/scroll props, validation, flash, redirects, asset versioning, Vite and SSR. Also covers the framework-independent Inertia protocol and client semantics: page objects, X-Inertia headers, partial reloads, Precognition and adapter conformance."
---

# Goravel Inertia

`github.com/goravel/inertia` integrates Inertia.js v3 with Goravel's HTTP lifecycle, sessions, validation and routing. This skill bundles both the adapter and the protocol it implements, so one install is enough.

## Versions

- **Adapter v1.18.0:** read [versions/v1.18.0.md](versions/v1.18.0.md) before configuring or using this release.
- **Inertia.js v3 protocol:** read [versions/inertia-v3.md](versions/inertia-v3.md) for page objects, request and response headers, prop categories, redirects and client behavior.

Identify the installed adapter version in `go.mod` before applying this guidance. If it is not listed, consult that version's tagged documentation and source; do not assume v1.18.0 behavior applies.

## Which reference to read

- Goravel code (controllers, middleware, config, tests): start with the adapter reference, then the protocol reference for the meaning of a header or a prop category.
- Protocol behavior or an adapter bug: read the protocol reference, then check what the installed adapter actually does.

When they disagree, the installed adapter wins for that project and the official docs win for the specification. The known divergence in v1.18.0 is flash: the protocol puts `flash` at the top level of the page object, while the adapter exposes `props.flash`. Precognition is specified by the protocol but not implemented in v1.18.0.
