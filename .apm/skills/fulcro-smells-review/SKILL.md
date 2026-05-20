---
name: fulcro-smells-review
description: Review Fulcro code against Fulcro-specific smells (placeholder, see TODO); pure report, never writes
argument-hint: "[path|all]"
allowed-tools: Bash, Read, Grep, Glob
user-invocable: true
disable-model-invocation: true
---

# Fulcro Smells Review (Placeholder)

This skill is a placeholder. Fulcro-specific code review against a curated smells catalog is planned but not yet implemented. Running `/fulcro-smells-review` today shows this notice and exits.

See `CONVENTIONS.md` in the repo root for the argument grammar this skill will follow once implemented.

## Arguments

| Input              | Target                                                                       |
|--------------------|------------------------------------------------------------------------------|
| (no argument)      | Review changed files only (staged + unstaged)                                |
| `all`              | Review the full codebase, sampling high-risk and high-traffic namespaces     |
| `<path>` `<glob>`  | Review files or directories matching the path                                |

Examples once implemented:

```
/fulcro-smells-review
/fulcro-smells-review src/main/app
/fulcro-smells-review src/main/app/ui/root.cljs
/fulcro-smells-review all
```

This skill is pure-report: it never writes. Operators apply suggestions themselves. No `--report` flag, because there is nothing to invert.

When no argument is supplied and the working tree is not a git worktree, the eventual implementation will ask the operator what to review rather than silently widening to `all`.

## Status

**Not implemented.** A Fulcro-specific smells catalog is pending. The host-neutral [clj-smells catalog](https://github.com/nufuturo-ufcg/clj-smells-catalog) covered by `/clj-smells-review` already catches cross-dialect smells in the `.clj`, `.cljc`, and `.cljs` files of a Fulcro project, but it does not flag the framework-specific failure modes that come from ident hygiene, query / state-shape drift, mutation section confusion, transit serialization hazards, and unnecessary re-renders.

For host-neutral smells, run `/clj-smells-review`. For ClojureScript-specific smells, watch for the future `/cljs-smells-review`. This skill will complement both with Fulcro-specific checks once the catalog is curated.

## Planned Categories

When implemented, this review will cover Fulcro-specific failure modes:

- **Ident hygiene**: components that mix table keywords for the same conceptual entity, ident values pulled from somewhere other than `props`, singletons that use a non-`:component/id` keyword, idents constructed by hand instead of through `(comp/get-ident ComponentName props)`.
- **Query and state-shape drift**: `:query` entries that are not produced by the component's `:initial-state` or by any registered Pathom resolver; components whose render reads keys not present in `:query`.
- **Direct `swap!` on the state atom**: button handlers that call `swap!` directly instead of going through `(comp/transact! this [(mutation params)])`; mutation `action` bodies that perform network work instead of declaring it under `remote`.
- **Mutation section confusion**: `action` bodies that contain network calls, `remote` bodies that mutate client state, `ok-action` bodies that re-execute the optimistic change, missing `error-action` on mutations that have a `remote`.
- **Transit serialization hazards**: mutation parameters or load payloads that contain atoms, channels, refs, functions, or arbitrary deftypes; Pathom resolver outputs with values that fail to round-trip through transit.
- **Missing `:initial-state`**: components that appear in a parent's `:query` but are never seeded into the normalized DB at boot, causing silent `nil` render until a load completes.
- **Callback-induced re-renders**: inline closures in props that change identity on every render; components that pass parent-derived data as bare props instead of through `(comp/computed factory props {...})`.
- **`comp/computed` omissions**: parent components that build a `(factory child)` call with callbacks but do not wrap the props in `comp/computed`, defeating the framework's identity-based render skip.
- **Legacy `defui` and union routers**: components defined with the deprecated `defui` macro, routers defined with the legacy union-table `defrouter` shape, code that imports from `fulcro.client.*` instead of `com.fulcrologic.fulcro.*`.
- **Over-querying**: components whose `:query` includes keys the render body never reads, causing wasted re-renders when those keys change.
- **Pathom 2 holdovers**: imports from `com.wsscode.pathom.connect` in projects whose other resolvers are on `com.wsscode.pathom3.connect.operation`; mixed Pathom 2 / Pathom 3 syntax in the same parser.
- **`with-redefs` on Fulcro internals**: tests that rebind framework hooks after `(app/fulcro-app ...)` has already read them, producing the appearance of a stub without the behavior.

## Tracking

See `TODO.md` in the repo root.

## Output

When invoked, print this notice and exit:

```
fulcro-smells-review is not yet implemented.

A Fulcro-specific smells catalog is in development. For now:
  - Run /clj-smells-review for host-neutral smells in .clj, .cljc, and .cljs files.
  - Run /fulcro-fix for lint, format, test, advanced-compilation, and duplicate-form checks.
  - The fulcro skill (auto-invoked) covers idiomatic Fulcro components, idents, queries, mutations, loads, routing, forms, UI state machines, and the Pathom 3 server.

To track progress, see TODO.md in the fulcro-skills repo.
```

Do not run any analysis. Do not invoke clj-kondo. Do not consult the host-neutral clj-smells catalog.
