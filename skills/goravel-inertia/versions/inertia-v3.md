# Inertia.js v3 protocol — reference

Inertia is one HTTP protocol, not a framework feature. The server owns routing,
data and auth; the client swaps page components without a full reload. Every
adapter implements the same contract, so anything below holds whatever the
server and client frameworks are.

Source of truth: <https://inertiajs.com/docs/v3/core-concepts/the-protocol>.
This reference ships with the Goravel adapter skill: read it for the contract and
[v1.18.0.md](v1.18.0.md) for the Goravel implementation. When the two disagree,
the installed adapter wins for that project and the docs win for the protocol.
Known divergence in v1.18.0: flash (see [Validation errors and
flash](#validation-errors-and-flash)).

## The page object

Both response modes carry the same object. Only `component`, `props`, `url` and
`version` always appear; the client reads a missing field as `[]`, `{}` or `false`.

```json
{
  "component": "User/Edit",
  "props": { "errors": {}, "user": { "name": "Jonathan" } },
  "url": "/user/123",
  "version": "6b16b94d7c51cbe5b1fa42aac98241d5"
}
```

Optional fields, emitted only when they apply: `encryptHistory`,
`clearHistory`, `preserveBigIntegers`, `preserveFragment`, `mergeProps`,
`prependProps`, `deepMergeProps`, `matchPropsOn`, `scrollProps`, `deferredProps`,
`rescuedProps`, `sharedProps`, `onceProps`, `liveProps`, `flash`.

`preserveBigIntegers` wraps integers beyond JavaScript's safe range as
`{"$bigint": "<digits>"}` markers; relevant when the server sends 64-bit values
such as money in minor units.

## Two response modes

A visit with no `X-Inertia` header is a full-page request and gets a complete
HTML document: assets, a root element, and the page object in
`<script data-page="app" type="application/json">`. The current client reads the
script element; it no longer reads a `data-page` attribute on the root `<div>`.

A visit that carries `X-Inertia: true` gets the page object as JSON, with
`Content-Type: application/json`, `X-Inertia: true` and `Vary: X-Inertia`. The
`Vary` header is required: one URL legitimately serves HTML and JSON.

The serialized page object must not contain a literal `</script>`, which would
close the element early. Escape `/` as `\/` or `<` as `\u003C`; HTML entities are
wrong here, since the browser does not decode them inside a script element and
JSON parsing fails.

## Request headers

| Header | Meaning |
| --- | --- |
| `X-Inertia: true` | Marks an Inertia visit |
| `X-Inertia-Version` | Asset version of the current page; omitted when empty |
| `X-Inertia-Partial-Component` | Component being reloaded; required for a partial reload |
| `X-Inertia-Partial-Data` | Comma list of props to include |
| `X-Inertia-Partial-Except` | Comma list of props to omit |
| `X-Inertia-Reset` | Comma list of prop paths the client resets before applying |
| `X-Inertia-Error-Bag` | Error bag for validation errors |
| `X-Inertia-Infinite-Scroll-Merge-Intent` | Append or prepend the requested page |
| `X-Inertia-Except-Once-Props` | Once-prop keys the client already holds unexpired |
| `Purpose: prefetch` | Prefetch request |
| `X-Socket-Id` | Broadcaster connection id, for live props |
| `X-XSRF-TOKEN` | CSRF token read from the `XSRF-TOKEN` cookie |
| `Precognition`, `Precognition-Validate-Only` | Precognition validation requests |

## Response headers and control responses

`X-Inertia: true` and `Vary: X-Inertia` mark a page response. A control response
is `409 Conflict` and carries no `X-Inertia` header:

- `X-Inertia-Location` — full-page `window.location` visit. Used for external
  redirects and for an asset version mismatch.
- `X-Inertia-Redirect` — the client makes a new Inertia `GET` visit instead.
  Used when the redirect target holds a URL fragment.
- `X-Inertia-Version` — the server's current version, set on a version mismatch.

A background request that hits a version mismatch keeps the current page instead
of navigating, so an unsaved form is not discarded.

## Redirects

Standard `3xx` redirects are followed by the browser, which re-sends
`X-Inertia`. The client inspects only the final response in the chain.

A `302` after `PUT`, `PATCH`, `DELETE` or `QUERY` must become `303 See Other`,
or the browser repeats the original method on the redirect target.

A redirect that leaves the Inertia app becomes `409` + `X-Inertia-Location`,
because the app's headers do not carry across origins. Prefetch requests are the
one exception and receive the redirect unchanged.

## Props: categories and evaluation

Every prop belongs to one or more categories. The category decides whether it
resolves for a request and which metadata field announces it.

| Category | Full visit | Partial reload | Metadata |
| --- | --- | --- | --- |
| Regular | Resolved | Only when `only`/`except` selects it | none |
| Always | Resolved, ignores `only`/`except` | Resolved, ignores filters | none |
| Optional | Skipped, not announced | Only when selected | none |
| Deferred | Skipped, announced | Only when selected | `deferredProps` (full visit only) |
| Merge / prepend / deep merge | Resolved | When resolved | `mergeProps`, `prependProps`, `deepMergeProps`, `matchPropsOn` |
| Once | Resolved unless the client holds it | When requested, ignoring except-once | `onceProps` |
| Scroll | Resolved | When resolved | `scrollProps`, `mergeProps` |
| Live | Resolved | When resolved | `liveProps` |

Categories compose: a prop may be deferred *and* mergeable, once *and* deferred,
scroll *and* deferred. Metadata fields stay independent.

`errors` is the classic always prop, shared by middleware, so every page object
carries it unless a page prop overrides it.

### Partial reloads

`X-Inertia-Partial-Data` narrows the response; `X-Inertia-Partial-Except` removes
from it. With both, narrow first, then remove — a prop in both is left out. Entries
may be dot paths (`auth.user`). A partial reload applies only to the component
named in `X-Inertia-Partial-Component`; a request that ends on another component
resolves as a full visit, which is what happens when the session expired and the
target is the login page.

A reset prop is sent in both `X-Inertia-Reset` and `X-Inertia-Partial-Data`, and
comes back without a merge label, so the client replaces rather than merges it.

```http
REQUEST
X-Inertia: true
X-Inertia-Partial-Component: Posts/Index
X-Inertia-Partial-Data: posts
X-Inertia-Reset: posts

RESPONSE 200
{ "component": "Posts/Index", "props": { "posts": [ ... ] }, "url": "/posts", "version": "..." }
```

No `mergeProps` entry for `posts` in that response: the client replaces the array.
For a scroll prop the server also sets `scrollProps.posts.reset`.

### Deferred and rescued

A deferred prop is skipped on a full visit and listed under `deferredProps`,
grouped by arbitrary request group; the client sends one partial reload per group.
Optional props are never announced and resolve only when selected.

A deferred prop that throws yields `200` with the rest of the props, leaves the
prop out entirely (not `null`) and adds its key to `rescuedProps`, which appears
only on partial reloads. The client renders the deferred component's `rescue`
state for it.

### Merging and infinite scroll

Merge metadata applies on partial reloads only: a full visit replaces the prop
even when it carries a label. `matchPropsOn` names the identifying field per
prop path, as `"<propPath>.<keyField>"` split at the last dot, so updates land
in place instead of duplicating items.

```json
{
  "props": { "posts": { "data": [{ "id": 11, "title": "First" }] } },
  "mergeProps": ["posts.data"],
  "matchPropsOn": ["posts.data.id"]
}
```

Read `posts.data.id` as prop path `posts.data` plus key field `id` (split at the
last dot): an incoming item with `id` 11 replaces the existing one instead of
appending a duplicate.

Infinite scroll builds on merge props: the inner array is labelled (`posts.data`)
and `scrollProps` reports `pageName`, `previousPage`, `nextPage`, `currentPage`
and `reset`. A deferred scroll prop emits no `scrollProps` on the full visit,
because it resolves in the follow-up request.

### Once and live

A once prop resolves on first use; the client reuses it across pages until it
expires or the server refreshes it. The client reports keys it already holds in
`X-Inertia-Except-Once-Props`; the server skips them, except on a partial reload
that requests them, or when it deliberately sends a new value.

A live prop refreshes on a broadcast event through a partial reload, or carries
new values under `__inertia.props` in the event payload.

## Validation errors and flash

Errors travel in the `errors` prop: `{}` when clean, otherwise each key mapping
to a message or an array of messages. `X-Inertia-Error-Bag` keeps several forms
on one page apart.

Flash is a top-level page object field in v3, included only when flash data
exists. The client exposes it through the `inertia:flash` event and leaves it out
of the saved history state. An adapter may instead share it as a `flash` prop:
`goravel-inertia` v1.18.0 does, so read `props.flash` there, never `page.flash`.
Assert the shape the installed adapter produces; a shared prop does not have the
same history semantics as the protocol's top-level flash.

## Precognition

Precognition validates a form without running the action. The client sends the
normal request plus `Precognition: true`, and `Precognition-Validate-Only` with a
comma list of field names when only some need checking. The server validates and
answers:

- `204 No Content` with `Precognition: true` and `Precognition-Success: true`
  when validation passes.
- `422 Unprocessable Entity` with the errors body when it fails.
- `Vary: Precognition` on every response that passes through the validation
  middleware.

The action must not execute: no writes, no flash, no redirect. The client shows
the errors inline and sends the real request later.

`github.com/goravel/inertia` v1.18.0 does **not** implement Precognition, and
neither does `petaki/inertia-go` v1.15.0. A Precognition request therefore reaches
the handler like any other and can run the mutation. Either add a middleware or
route guard that validates `Precognition-Validate-Only` and returns `204`/`422`
before the handler, or leave Precognition disabled on the client in that project.

## Status codes

| Code | Use |
| --- | --- |
| `200` | Page response, HTML or JSON |
| `302` | Redirect after `GET`/`POST` |
| `303` | Redirect after `PUT`/`PATCH`/`DELETE`/`QUERY` |
| `409` | Control response: version mismatch, external redirect, fragment redirect |
| `204` / `422` | Precognition validation passed / failed |

## Client surface

Names differ per client package; the contract does not: `Link` for navigation,
`router.visit/reload/replaceProp`, a form helper (`Form` component or `useForm`)
that owns errors, processing state and progress, `Deferred`, `WhenVisible`,
`InfiniteScroll`, prefetching, and the poll helper.

## Adapter work

An adapter is correct when it does all of this:

1. Detects `X-Inertia` and answers JSON instead of the HTML document.
2. Sets `Vary: X-Inertia` on both modes.
3. Emits the HTML page object inside `data-page`, escaped for `</script>`.
4. Filters props by category and mode, and emits the matching metadata.
5. Compares `X-Inertia-Version` and answers `409` on a `GET` mismatch, keeping
   flash for the next request; non-`GET` requests never receive it.
6. Turns `302` into `303` after `PUT`/`PATCH`/`DELETE`, and answers `409` with
   `X-Inertia-Location` for redirects that leave the app.
7. Shares `errors` as an always prop and honours the error bag header.
8. Serves the root template with the page object and the client entry, and
   exposes the asset version that goes into the page object.
9. Answers Precognition requests before the action: `204` on success, `422` with
   errors on failure, and never runs the mutation.

Test at the seam: send the real headers and assert component, props, status,
redirect target, `props.errors` and the flash shape. That covers pages and
mutations without mounting the client, and it is where protocol regressions
actually show up.

## Pitfalls

- Dropping `Vary: X-Inertia`, so a cache serves JSON to a browser or HTML to
  the client.
- `302` after a delete, so the browser re-sends `DELETE`.
- Announcing deferred props on a partial reload, or resolving them on the full
  visit merely because they look cheap.
- Treating a partial reload for a different component as partial.
- Reading `props.flash` on an adapter that emits top-level `flash`, or the reverse.
- Returning a `409` for a non-`GET` request.
- Sending `null` for a deferred prop that failed instead of listing it in
  `rescuedProps`.
- Enabling client-side Precognition on an adapter that ignores its headers, so a
  validation request executes the action.
