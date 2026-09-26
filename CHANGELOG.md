# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.13] - 2026-09-26

### Fixed

- `fulcro-new` now scaffolds the `:test` alias with `fulcrologic/fulcro-spec` 3.2.10. The previous coordinate, `com.fulcrologic/fulcro-spec` 3.1.31, does not exist on Clojars, so `clj -M:test` failed in every generated project. The `fulcro` skill's project-workflows reference used the same nonexistent version and now also names 3.2.10.
- `fulcro-new` now asks for dependency versions when the Clojars lookup fails, matching `fulcro-upgrade`, instead of falling back to a stale Fulcro 3.8.x hint.
- `fulcro-fix` now reports how many vendored or generated paths it skipped, and lists them under `--report`, as Rule 4 requires.
- Skills no longer point at `CONVENTIONS.md` "in the repo root", which is absent from the projects where the skills run, or at `TODO.md`, which is not published. The `fulcro-smells-fix` notice no longer links to `TODO.md`.

## [0.1.12] - 2026-09-16

### Changed

- The README's runtime sentence now names Grok Build after Kiro. APM 0.28.0 added `grok-build` as a canonical target that `apm install --target all` includes and that APM auto-detects from a `.grok/` directory, so the default target set now has nine runtimes. Grok Build keeps a target-native skill directory (`.grok/skills/` at project scope, `~/.grok/skills/` at user scope), like Claude Code and Kiro, rather than the shared `.agents/skills/` directory.

## [0.1.11] - 2026-07-29

### Changed

- The README's runtime sentence now scopes its list to APM's default target set rather than claiming every runtime APM supports. APM 0.26.0 also supports Antigravity, IntelliJ, and several experimental runtimes, none of which `apm install --target all` includes. The eight runtimes named are unchanged, and the sentence adds that Antigravity works when named explicitly with `--target antigravity`.

## [0.1.10] - 2026-07-29

### Removed

- The package manifest no longer declares the top-level `target: all` field. The APM manifest schema deprecates the `all` value: a parser treats the field as though it were absent and falls through to the `--target` flag or filesystem auto-detection, and the value is scheduled to become a hard parse error in a future APM release. Removing the field makes that fall-through behavior permanent. Installation behavior is unchanged, because APM already resolved targets by auto-detection rather than from this field. The separate `compilation.target` setting is not affected.

## [0.1.9] - 2026-06-23

### Changed

- The `fulcro-fix` dry step is now advisory and scoped to production source, matching `clj-fix`. It scans the source paths declared by the project's build configuration (`deps.edn` `:paths` or `shadow-cljs.edn` `:source-paths`) rather than a hardcoded `src test`, reads dry4clj's `--edn` output instead of matching a status string, and reports candidates without failing the run or halting the pipeline. The step reports `PASS` when no candidates are found, `REVIEW` when it lists candidates for inspection, `ERROR` when the scan cannot run or parse, or `SKIPPED`. Test directories are excluded by default because repeated test scaffolding is usually intentional.
- `CONVENTIONS.md` Rule 3 now states that mutating skills apply safe edits and report findings that require judgment, and that `--report` suppresses intentional source and configuration edits while verification steps such as the advanced release build may still write build artifacts.

## [0.1.8] - 2026-06-15

### Changed

- Add Kiro to the README's runtime list. APM 0.20.0 added Kiro as a first-class install target included in `apm install --target all`, so the README now lists it alongside Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, and Windsurf.

## [0.1.7] - 2026-05-28

### Added

- `CONVENTIONS.md` gains Rule 4: vendored and generated paths are excluded by default from mutating skills that walk the workspace. The rule sits alongside the existing three rules (renamed from "The three rules" to "The four rules"). Two filters apply together (`.gitignore` matches plus a hardcoded floor of dependency directories, build outputs, and lock files). The override rides on Rule 1's existing `<path>` `<glob>` grammar; no new flag is introduced. Applies to `/fulcro-fix` and `/fulcro-smells-fix`. Exempt: `/fulcro-new` is scaffolding, `/fulcro-upgrade` rewrites Fulcro dependency declarations. The `advanced` step of `/fulcro-fix` produces shadow-cljs release output that is outside the source-mutation scope of the rule.
- `/fulcro-fix` and `/fulcro-smells-fix` each extend their `## Mutation` (or placeholder body) section with a paragraph restating Rule 4 in context. Operators see the policy without leaving the skill's page.

## [0.1.6] - 2026-05-26

### Fixed

- Quote the YAML `description` and `argument-hint` frontmatter in all skills that had unquoted values. Prevents potential YAML misinterpretation of special characters (angle brackets in `argument-hint`, semicolons and parentheses in `description`).

## [0.1.5] - 2026-05-22

### Changed

- The `fulcro-smells-review` placeholder is renamed to `fulcro-smells-fix` to match the mutating contract adopted by the new `clj-smells-fix` skill in [clojure-skills](https://github.com/brackendev/clojure-skills). The placeholder still prints a "not yet implemented" notice; the rename, argument grammar (`[path|all] [--report]`), and frontmatter all reflect the eventual fix-by-default behavior. The "not yet implemented" notice now points operators at `/clj-smells-fix` for host-neutral smells. Operators with a saved `/fulcro-smells-review` invocation should replace it with `/fulcro-smells-fix`.

## [0.1.4] - 2026-05-20

### Changed

- `CONTRIBUTING.md` is aligned to the family-wide structural template. A `CONVENTIONS.md` row is added to the Layout table, step 1 of "Adding or modifying a skill" no longer carries an inline `CONVENTIONS.md` reminder (the dedicated Skill conventions section now opens with that pointer), and the Skill conventions section begins with the canonical pointer paragraph.

## [0.1.3] - 2026-05-20

### Changed

- The `fulcro-tidy` skill is renamed to `fulcro-fix` to adopt the noun-first canonical naming pattern (`<target>-<verb>`) shared across the agent-skills family. The verb suffix `-fix` consistently signals a mutating quality pipeline (lint, format, test, advanced, dry). Operators with a saved `/fulcro-tidy` invocation should replace it with `/fulcro-fix`. The skill's behavior is unchanged; only the name moves.
- Cross-references to `clojure-skills` are updated for the `clj-tidy` → `clj-fix` rename. The `/fulcro-new` "Next steps" footer now points at `/clj-fix` and `/fulcro-fix`; the `/fulcro-upgrade` recommended-next-steps and sanity-check comments now point at `/fulcro-fix advanced`.

## [0.1.2] - 2026-05-19

### Added

- A repo-root `CONVENTIONS.md` that defines the argument grammar, scope vocabulary, and mutation defaults every user-invocable skill in this package follows. Three rules cover argument grammar (one sanctioned flag, `--report`), scope vocabulary (`(no argument)`, `all`, `<path>`), and mutation-as-default. The document lists `/fulcro-new` as the standard's positional-required exemption and includes an author checklist that runs against every migrated skill.

### Changed

- The `fulcro-check` skill is renamed to `fulcro-tidy`. The verb now matches the default behavior: `cljfmt fix` runs by default and rewrites files in the `format` step. Operators with a saved `/fulcro-check` invocation should replace it with `/fulcro-tidy`. The new `--report` flag swaps the format step for `cljfmt check`, which previews diffs without writing; `lint`, `test`, `advanced`, and `dry` are pure-read of source regardless. The `advanced` step still writes build output under `resources/public/js/` (shadow-cljs release), which is a build artifact, not source. Breaking for saved `/fulcro-check` invocations.
- The `fulcro-new` skill gains a `## Arguments` section that documents its positional `<project-name>` exemption and a `## Mutation` section that lists the files it writes. The "Next steps" footer in the scaffold output now points at `/fulcro-tidy` and `/clj-tidy` instead of the renamed-away `/fulcro-check` and `/clj-check`.
- The `fulcro-upgrade` skill gains a `## Arguments` section, a `## Mutation` section, and a `--report` flag. With `--report`, the skill prints the current and latest released versions for each Fulcro artifact found in `deps.edn` and surfaces the upstream `CHANGELOG.adoc` for major version jumps without writing `deps.edn`, refreshing dependencies, or running the compile and tests. The skill's recommended-next-steps and sanity-check comments now point at `/fulcro-tidy` rather than the renamed-away `/fulcro-check`.
- The `fulcro-smells-review` placeholder gains a canonical `## Arguments` section using the core scope vocabulary (`(no argument)`, `all`, `<path>`) and an explicit pure-report classification ahead of the eventual implementation. The "not yet implemented" notice now points at `/fulcro-tidy` instead of `/fulcro-check`.

## [0.1.1] - 2026-05-18

### Fixed

- Remove Claude Code-specific `<package>:` namespace prefix from slash commands referenced inside skill bodies. The neutral form (`/fulcro-check`, `/fulcro-new`, `/fulcro-smells-review`, `/clj-check`, `/clj-smells-review`, `/cljs-smells-review`) resolves correctly on every runtime APM deploys to, matching the form already used in `README.md`.

## [0.1.0] - 2026-05-17

### Added

- Initial release. Fulcro full-stack framework skills layered on top of [clojure-skills](https://github.com/brackendev/clojure-skills) for host-neutral style, [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) for the ClojureScript client, and [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) for the JVM Clojure server. Install all four together for Fulcro projects.
- **fulcro** (model-invoked): `defsc` component definition with co-located `:query`, `:ident`, `:initial-state`, and render body; idents and the normalized client database (`[:table/id value]`, `[:component/id ::ComponentName]`, hygiene rules); EQL queries (joins, parameterized joins, link queries, recursive queries); mutations with `action` / `remote` / `ok-action` / `error-action` / `result-action` sections; `df/load!` and `df/load-field!` with targeting helpers, markers, and post-mutations; dynamic routing (`defrouter`, `dr/change-route!`, `:route-segment`, `dr/route-deferred`, `dr/target-ready`); form-state algorithms (`fs/add-form-config*`, `fs/dirty?`, `fs/validity`, `fs/mark-complete*`); UI state machines (`com.fulcrologic.fulcro.ui-state-machines`); the Pathom 3 server resolver and mutation shape (`pco/defresolver`, `pci/register`, `p.eql/process`); and the Fulcro Inspect workflow (Chrome extension, `:preloads` for `com.fulcrologic.fulcro.inspect.preload` and `com.fulcrologic.fulcro.inspect.dom-picker-preload`, DB / Transactions / Network / EQL tabs). Defers to the [clojure](https://github.com/brackendev/clojure-skills) baseline for host-neutral style, the [clojurescript](https://github.com/brackendev/clojurescript-skills) skill for JavaScript interop and the CLJS workflow, and the [clojure-jvm](https://github.com/brackendev/clojure-jvm-skills) skill for Java interop on the server and the JVM tooling workflow. Pulls deeper detail from `references/project-workflows.md` on demand (project layout, `deps.edn`, `shadow-cljs.edn`, `dev/user.clj`, the development loop, production build, Pathom 3 wiring, the test stack, the `guardrails` toggle, and the surrounding ecosystem).
- **fulcro-lenses** (model-invoked): Code-lenses translations layered on top of `clojure-lenses` (in `clojure-skills`) and `clojurescript-lenses` (in `clojurescript-skills`). Records only the Fulcro-specific deltas: the normalized client database, ident composition, mutations as data, EQL queries, the `action` / `remote` / `ok-action` / `error-action` split, `df/load!` over hand-rolled merges, dynamic routing, form-state algorithms, UI state machines, the Pathom 3 server, and the Fulcro Inspect workflow. Covers the four default code-lenses philosophies (grug, Honest Code, Tidy First, Parse Don't Validate) and the two opt-in philosophies (APOSD, Legacy Code) that activate when their lens is added with `+aposd` or `+legacy-code` or invoked directly. Auto-triggers alongside the [code-lenses](https://github.com/brackendev/code-lenses) plugin in Fulcro work.
- **fulcro-new** (user-invoked): Scaffold a new Fulcro full-stack project. Creates `deps.edn` with `:dev`, `:test`, `:cljfmt`, `:build`, `:dry4clj` aliases; `shadow-cljs.edn` with `:main`, `:test`, and `:ci` (Karma) builds plus Fulcro Inspect preloads; `package.json` with shadow-cljs, react, react-dom, and karma dependencies; client entry files (`app.application`, `app.client`, `app.ui.root`, `app.model.session`); server files (`app.server.main`, `app.server.pathom`, `app.server.middleware`) wired through a `/api` transit endpoint; `dev/user.clj` with `(start)` / `(stop)` / `(restart)` helpers; `.clj-kondo/config.edn` with `:lint-as` entries for Fulcro and Pathom macros plus upstream import bootstrap; `.cljfmt.edn` with Fulcro-aware indentation; production and development HTML shells; and `.gitignore`. Fetches the latest released versions of `com.fulcrologic/fulcro`, `com.fulcrologic/guardrails`, and `com.wsscode/pathom3` from Clojars before writing `deps.edn`.
- **fulcro-check** (user-invoked): Run the Fulcro quality pipeline (lint, format, test, advanced, dry). Lint bootstraps upstream Fulcro and Pathom clj-kondo exports with `clj-kondo --copy-configs --dependencies` if missing, then runs `clj-kondo --lint src test`. Format runs `clj -M:cljfmt fix`. Test aggregates server (Kaocha via `clj -M:test`) and client (shadow-cljs `:test` build, with Karma fallback) suites and reports per-side status. Advanced runs `npx shadow-cljs release main` as a production-build sanity check that surfaces externs-inference failures and Closure-renaming warnings. Dry runs `clj -M:dry4clj src test` and passes only on the literal `No duplicate candidates found.` line. Each step is also addressable on its own.
- **fulcro-upgrade** (user-invoked): Upgrade the `com.fulcrologic/fulcro` dependency in `deps.edn` to the latest released version on Clojars. Optionally bumps `com.fulcrologic/fulcro-rad` and `com.fulcrologic/guardrails` if present. Records current versions, fetches latest from Clojars' `latest_release` endpoint, surfaces the upstream `CHANGELOG.adoc` for major version jumps before applying, updates `deps.edn`, refreshes dependencies with `clj -P`, runs compile (`npx shadow-cljs compile main`) and test (`clj -M:test` plus the client suite) to validate, and refreshes clj-kondo imports. Reverts the version on compile or test failure.
- **fulcro-smells-review** (user-invoked, placeholder): Reserves the command name for a future Fulcro-specific smells review. Currently prints a "not yet implemented" notice that points users at `/clj-smells-review` in `clojure-skills` for host-neutral smells in `.clj`, `.cljc`, and `.cljs` files. Documents the planned categories: ident hygiene, query / state-shape drift, direct `swap!` bypassing the transaction layer, mutation-section confusion, transit serialization hazards, missing `:initial-state`, callback-induced re-renders, `comp/computed` omissions, legacy `defui` and union-router holdovers, over-querying, Pathom 2 holdovers, and `with-redefs` on Fulcro internals.
