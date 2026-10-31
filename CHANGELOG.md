# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.26] - 2026-10-31

### Changed

- The Honest Code section of `fulcro-lenses` cites the Honest Code Principles by number (P01-P25), which the code-lenses `honest-code` skill now follows, instead of the construct names code-lenses no longer uses. It reports the normalized client database as a departure from P05 and lifecycle effects as a departure from P22, instead of describing them as conforming, and it no longer suggests that `with-redefs` stubbing is a reason to keep state in the database.

## [0.1.25] - 2026-10-29

### Added

- `fulcro-rules` hook: delivers the condensed FUL-1 through FUL-5 Iron Laws to the agent at session start, after `/clear` or a compaction, and on the first matching edit when the session started outside a matching project, with an instruction to load the `fulcro` skill along with the `clojure`, `clojurescript`, and `clojure-jvm` skills. Verified only under Claude Code. Copilot and Cursor receive no copy.
- Hook test suites in `tests/hooks/` (`bb tests/hooks/run.bb`), and empty `copilot-hooks.json` and `cursor-hooks.json` files.

## [0.1.24] - 2026-10-29

### Changed

- The Install section requires APM 0.34.0 and installs it from PyPI with `uv tool install apm-cli==0.34.0` after removing any Homebrew copy. APM 0.35.0 is not supported because it can delete installed skill files ([microsoft/apm#3372](https://github.com/microsoft/apm/issues/3372)).

## [0.1.23] - 2026-10-25

### Added

- The `fulcro` project workflows reference adds the files a project needs for its documented commands to work: a `package.json` with React 19.3.1, a `tests.edn` that points Kaocha at `src/test`, and a `karma.conf.js` for the `:ci` client test build. Without `tests.edn`, `clj -M:test` found no tests and still exited 0.
- The project workflows reference proxies the dev HTTP server's `/api` requests to the Jetty server, enables guardrails in ClojureScript dev builds through the compiler's external configuration, and shows `--force-spawn` for a release build that must not run inside a running shadow-cljs server.
- The Pathom server reference shows how to give a server mutation the client's symbol with `::pco/op-name`, and how to keep tempid remapping working with `m/returning`: Pathom 3 omits `:tempids` from a mutation join unless the query asks for it, so the reference adds an optional `:tempids` through the app's `:global-eql-transform`.
- The mutations reference covers `m/returning` and `m/with-target` for merging a mutation's return value, and custom transit types with `transit/install-type-handler!`.
- The routing reference lists the route lifecycle rules: route explicitly at startup, keep `:will-enter` free of side effects, and guard leaving a route with `:allow-route-change?` and `:route-denied`.
- The state machines reference explains that handlers return the updated `env` and use `uism/apply-action`, `uism/trigger-remote-mutation`, `uism/load`, and `uism/transact` instead of calling `comp/transact!`.
- The components reference shows how to rebuild a component's props from the client database at the REPL when its props are unexpectedly `nil`, following Jakub Holý's Fulcro Troubleshooting Decision Tree.
- `/fulcro-fix` covers the `fulcro-spec` macros in its cljfmt guidance, shows the clj-kondo configuration that `fulcro-spec` tests need, and reports a Kaocha run that finds no tests as a failure.

### Changed

- The skills describe only Fulcro 3.10.0 and Pathom 3. Guidance for Pathom 2, union routers, `defui`, the Inspect preload that only logs "Inspect NOT installed", and notes about what individual Fulcro 3.8 and 3.9 releases changed are removed. The README states that Pathom 3 is published only as alpha releases, so the skills target the newest one.
- `/fulcro-upgrade` reads the upstream `CHANGELOG.md` and the commit history between the project's release tag and the latest release tag for Fulcro, Fulcro Inspect, guardrails, and Fulcro RAD at run time, and lists the places in the project that the reported changes affect. It replaces the fixed notes for Fulcro 3.8, 3.9, and 3.10.
- `/fulcro-upgrade` upgrades only to stable releases. When Clojars reports a release candidate as the latest release, the skill picks the newest stable version or asks.
- The README requires Java 21, which shadow-cljs needs to build the Fulcro client, and names the Chrome Web Store as a source for the Fulcro Inspect extension. The store listing carries the same 4.1.0 release as the fulcro-inspect releases page.
- The ecosystem table no longer lists `fulcro-incubator`, which depends on Fulcro 2.
- The project workflows reference pins Logback 1.6.6, the latest release.
- `/fulcro-fix` formats with the first cljfmt runner the project supports: a `:cljfmt` alias, the `lein-cljfmt` plugin, a `cljfmt` binary on `PATH`, or a `cljfmt` Clojure CLI tool. When none is installed, the format step reports that it was skipped and shows the `:cljfmt` alias to add, instead of failing and stopping the pipeline. The step also explains which configuration files cljfmt reads and how to pass paths to each runner.
- The `:cljfmt` alias in the project workflows reference and in the `/fulcro-fix` hint uses `:replace-deps`, so a project dependency that conflicts with cljfmt or cannot be resolved does not stop formatting.
- The `/fulcro-fix` signature lists the path and glob arguments that its arguments table already accepted: `/fulcro-fix [lint|format|test|advanced|dry] [--report] [all|<path>|<glob>]`.
- `/fulcro-upgrade` also upgrades `fulcrologic/fulcro-spec`, updates its `:dev` and `:test` entries together, and reads the fulcro-spec changelog and the commits between its release tags.
- `/fulcro-upgrade` reads the upstream changes before it writes `deps.edn` and before `--report` stops. Under `--report`, it also skips an artifact that is already at the latest stable version, never proposes a lower version, and prints a `Changes:` line that lists the changes that affect the project with their file matches. The final report carries the same line.
- `/fulcro-upgrade` restores the saved `deps.edn` after a dependency, compile, or test failure, and its dependency refresh downloads the aliases that declare the upgraded artifacts.
- Every command that the skills tell the agent to run starts with `clojure` instead of `clj`, for example `clojure -M:test` and `clojure -T:build uber`. The `clj` launcher exits with an error when `rlwrap` is not installed. The project workflows reference keeps `clj` for the two commands that start an interactive REPL.
- The `fulcro` skill's trigger description fits the 1,024-character limit of the Agent Skills specification. It keeps every trigger term and leaves out the summary of covered topics, which the skill body already describes.

### Fixed

- The mutation example reads the server's return value from `mutation-return-value`. `result` in `ok-action` is the network result map with `:status-code` and `:body`, so `(:user result)` was always `nil`.
- The mutation sections table explains that a custom `result-action` replaces the default result handling, so it must call `m/default-result-action!` or tempid rewriting, return-value merging, `ok-action`, and `error-action` stop working.
- The loading reference reads load markers through the `[df/marker-table marker-id]` query and `df/loading?`. It named `df/get-loading-state`, which does not exist.
- The forms reference and `fulcro-lenses` name the form-state validation API that exists, `fs/make-validator` and `fs/get-spec-validity`. They named `fs/validity`, `fs/form-field-valid?`, and `fs/valid?`, which do not exist.
- `fulcro-lenses` no longer claims that `m/declare-mutation` validates mutation parameters or that Fulcro RAD regenerates and overwrites project source.
- The project workflows reference adds `fulcro-spec` to the `:dev` alias, because shadow-cljs reads only that alias and the client test builds failed with "The required namespace "fulcro-spec.core" is not available".
- The guardrails section enables ClojureScript checks the way the guardrails README describes. It said to set the flag under `:jvm-opts` in `shadow-cljs.edn`, which shadow-cljs ignores when it reads dependencies from `deps.edn`.
- The middleware section no longer contradicts its own example about CSRF protection, and it shows how to send the anti-forgery token from the client while keeping the transit request encoding.
- The Fulcro Inspect reference describes the EQL tab as running queries through the app's remotes and lists the DB Explorer and Statecharts tabs.
- `/fulcro-fix` and `/fulcro-upgrade` detect client test builds by their `:karma` or `:node-test` target. They ran Karma for any `:test` build and fell back to a Node command that a browser test build cannot run.
- `/fulcro-fix` describes how `karma-chrome-launcher` finds Chrome and how a running shadow-cljs server affects a release build, and no longer claims that Fulcro's own source uses cljfmt's `:cursive` indentation.
- `/fulcro-upgrade` names the Fulcro RAD companion libraries by their Clojars coordinates (`com.fulcrologic/fulcro-rad-datomic`, `fulcro-rad-sql`, `fulcro-rad-semantic-ui`) and no longer claims that Clojars `latest_release` can name a SNAPSHOT build.
- `/fulcro-upgrade`, its description, and its runtime metadata say that the skill upgrades to the latest stable version. They said the latest released version, which can be a release candidate. The skill picks the stable version by comparing version numbers and skips SNAPSHOT builds and every prerelease form on Clojars, such as `3.9.0-rc14`.
- The `/fulcro-fix` dry step reports `SKIPPED`, the term that the other steps and the summary use. It said `SKIP`.
- `/fulcro-fix` recommends the cljfmt configuration that matches Fulcrologic's own code: `:function-arguments-indentation :cursive` with the catch-all `:indents {#re ".*" [[:inner 0]]}`, which indents every form body and every `(:require` libspec by two spaces. The configuration it recommended before rewrote much of Fulcro's own source and most `fulcro-template` files. The skill also explains that Fulcro's own `.cljfmt.edn` writes the pattern as `#".*"`, which cljfmt cannot read from an `.edn` file, and it keeps the `:extra-indents` form for projects that use the community indentation.
- The code examples in the `fulcro` references follow that configuration, so `cljfmt check` passes on code copied from them. The `ns` forms indent their libspecs by two spaces, and calls whose arguments continue on the next line indent them by two spaces.
- The dry4clj alias that `/fulcro-fix` shows lines up its keys inside their maps.

## [0.1.22] - 2026-10-25

### Added

- The `fulcro` skill opens with Iron Laws (FUL-1 through FUL-5): numbered rules that prevent stale UIs, silent data loss, and security holes. The agent does not write code that breaks a law, and when existing code breaks one, it reports the law ID, the location, and the correct pattern, then asks before changing code outside the task. Key Rules that guarded against bugs or security holes moved into the laws, and the remaining Key Rules are renumbered.
- The `fulcro` Pathom server reference covers securing the API: authentication in middleware, authorization in each resolver and mutation, and the Fulcro Developer's Guide's security chapter.

### Fixed

- The Pathom server reference attributes "You cannot trust a client" to the Fulcro Developer's Guide's "Full-Stack Error Handling" section, where it appears.

## [0.1.21] - 2026-10-25

### Changed

- The `fulcro` skill loads a short entry file that keeps the Key Rules and routes each task to a reference file: components and queries, mutations and loading, routing, forms, UI state machines, the Pathom server, and Fulcro Inspect. The detailed guidance moved into those files without changes, so less text loads on every matching task and the agent reads the detail for the topic at hand.

## [0.1.20] - 2026-10-25

### Changed

- When a project's `.cljfmt.edn` lists the Fulcro macros under `:indents`, `/fulcro-fix` now shows the `:extra-indents` configuration to use in its place.
- The Companion packages table describes biff-skills as covering Biff 2 conventions and deployment, because that package no longer scaffolds projects.

### Removed

- The `/fulcro-new` scaffolding skill is removed. The `fulcro` skill and its project workflows reference still describe the project layout, the `deps.edn` aliases, and the Fulcro Inspect setup.

## [0.1.19] - 2026-10-24

### Added

- The Companion packages table lists [electric-skills](https://github.com/brackendev/electric-skills), the new companion package for the [Electric](https://github.com/hyperfiddle/electric) v3 full-stack framework, and `CONTRIBUTING.md` cites its Hyperfiddle lavender brand color (`#B7A4C2`).
- The `fulcro` skill explains that a release build keeps Fulcro's Timbre log calls together with the absolute path of the Fulcro jar on the build machine, and its project workflows reference shows how to remove them at compile time: set `TAOENSSO_TIMBRE_MIN_LEVEL_EDN`, block the compiler cache for Timbre in release builds, and build in a fresh JVM.

### Changed

- The Companion packages table names the major version each framework package targets: Biff 2, Electric v3, and Fulcro 3. The skills in those packages do not apply to other major versions.
- The README introduction names Fulcro 3.
- When a project has no `:dry4clj` alias, `/fulcro-fix` reports the dry step as `SKIP` and shows the alias to add, instead of reporting `ERROR`.
- `CONTRIBUTING.md` asks for a changelog entry and a version increment for every change to a published file, including documentation-only changes, which matches the release history. It asked for entries only for user-facing changes.
- The `fulcro-lenses` description names the Fulcro namespaces and macros that trigger it, its layering, and the lenses it covers, without repeating the topic list from the skill body. Runtimes load every skill description on each request, so the shorter description costs less context.
- `/fulcro-new` pins shadow-cljs 3.5.7, React 19.3.1, and ring-jetty-adapter 1.15.6, the releases current on 2026-10-24, and the `fulcro` project workflows reference matches them. A project generated from the skill passed its compile, test, release, formatting, and lint checks with these versions.

### Removed

- The `/fulcro-smells-fix` placeholder is removed. It only printed a not-implemented notice. Run `/clj-smells-fix` from clojure-skills for host-neutral smells in `.clj`, `.cljc`, and `.cljs` files.

### Fixed

- The `CONTRIBUTING.md` Layout table names the local `AGENTS.md`, `GEMINI.md`, and `TODO.md` files. It named `CLAUDE.md`, which the package does not use.
- The README Requirements name the scaffolding command `/fulcro-new`. They named `/fulcro-skills:fulcro-new`, a plugin-style name that does not exist, because APM installs the package as skills with no package prefix.
- `/fulcro-fix` reads source paths from `deps.edn` when `shadow-cljs.edn` sets `:deps`, because shadow-cljs then ignores its own `:source-paths`. The `/fulcro-new` scaffold sets `:deps`.
- The README explains how Codex, OpenCode, and Gemini CLI offer the commands: Codex runs a skill as `$name` from its `/skills` list, OpenCode attaches one with `@name`, and Gemini CLI does not read the user-only setting. It said every command appears as a slash command and that user-only commands run only when typed.
- `CONTRIBUTING.md` states that `agents/openai.yaml` sets `allow_implicit_invocation` to `true` for every skill the model can run, including slash commands that the model can also run. It named only model-invoked and user-only skills.

## [0.1.18] - 2026-10-24

### Fixed

- The `fulcro` skill and `/fulcro-smells-fix` write `comp/computed` as `(comp/computed props {...})`, the signature Fulcro defines. The `fulcro` skill explains that computed props survive a targeted update, which drops callbacks passed in ordinary props.
- The `fulcro` skill allows `swap!` on the state atom in `ok-action`, `error-action`, and `result-action`, as its own mutation example does, instead of only in `action`.
- `/fulcro-fix` creates `.clj-kondo/` and copies the Fulcro and Pathom clj-kondo exports before every lint. Its earlier check looked for `.clj-kondo/.cache/imported`, a path clj-kondo never writes, and clj-kondo copies configurations only into an existing `.clj-kondo/` directory.
- `/fulcro-upgrade` skips and reports the clj-kondo imports refresh when the project has no `.clj-kondo/` directory.
- The `fulcro` skill, `fulcro-lenses`, and `/fulcro-smells-fix` say that Fulcro 3 has no `defui` macro and that legacy union routers remain available as `defsc-router`, which upstream supports but advises against in new applications.

### Changed

- The `fulcro` skill states each rule once. Its Gotchas for ident hygiene, `swap!` from UI code, inline callbacks, transit serialization, legacy forms, query drift, `remote` versus an `ast`, and docstring position are now part of the rules they repeated. `/fulcro-fix`, `/fulcro-new`, `/fulcro-upgrade`, and `fulcro-lenses` follow the same pattern.

## [0.1.17] - 2026-10-24

### Removed

- OpenCode 1 is no longer supported. The repository no longer keeps a copy of each `SKILL.md` under `.opencode/skills/` or a per-skill allow list in `opencode.jsonc`. `.opencode/skills` is now a symlink to `.apm/skills/` instead, so an OpenCode 2 session started anywhere in the repository loads the canonical skills together with their supporting files. The `.opencode/package.json` file, which pinned the OpenCode 1 plugin package, is removed as well. APM deploys the same files as before.

### Changed

- The `CONTRIBUTING.md` acceptance steps list the skills OpenCode discovers with `opencode api GET "/api/skill?location[directory]=$PWD"`, because OpenCode 2 has no `--pure` flag or `debug skill` command.

## [0.1.16] - 2026-10-24

### Fixed

- The `CONTRIBUTING.md` acceptance steps install with `apm install --target all` in the clean temporary project. On APM 0.34.0, a bare `apm install` in that project stops with "Multiple harnesses detected" because the steps create several runtime folders first. `apm update` reuses the targets that the install records in the project's `apm.yml`.

## [0.1.15] - 2026-10-17

### Changed

- In Claude Code and Codex, the agent can now run `/fulcro-fix` from a natural-language request, such as "run the Fulcro checks", as well as from the slash command. Its description now states when to use it. `/fulcro-new`, `/fulcro-upgrade`, and `/fulcro-smells-fix` remain user-only.

## [0.1.14] - 2026-10-04

### Fixed

- The Fulcro Inspect setup in the `fulcro` skill, its project-workflows reference, and the `fulcro-new` scaffold now works with Fulcro 3.8 and later. The previous `com.fulcrologic.fulcro.inspect.preload` preload only logs "Inspect NOT installed" on current Fulcro. Projects now add the `com.fulcrologic/fulcro-inspect` development dependency, use the `com.fulcrologic.devtools.chrome-preload` (or `electron-preload`) preload, and call `(ido (it/add-fulcro-inspect! app))` at startup. The extension or Electron app comes from the fulcro-inspect releases page instead of the Chrome Web Store.
- `fulcro-new` now writes `:deps {:aliases [:dev]}` in `shadow-cljs.edn`. With the previous `:deps true`, shadow-cljs could not see shadow-cljs itself or the other libraries declared in the `:dev` alias.
- `fulcro-new` and the project-workflows reference now declare the `:dry4clj` alias with a Git coordinate. The previous `dry4clj/dry4clj` 0.1.1 coordinate does not exist on Clojars, so the alias could not resolve.
- `fulcro-upgrade` no longer fetches the nonexistent upstream `CHANGELOG.adoc`. It reads the commit history between Fulcro release tags instead, because the replacement `CHANGELOG.md` stops at 3.7.2.
- The `fulcro` skill's main `defsc` example no longer refers to an unbound `props` inside its `:ident` lambda.
- `fulcro-new` now writes a `.cljfmt.edn` that keeps cljfmt's default indentation rules. The previous file listed Fulcro macros under `:indents`, which current cljfmt treats as a replacement for every default rule, so `cljfmt check` reported most generated files as incorrectly formatted and `/fulcro-fix` rewrote them. The new file uses `:extra-indents`, includes the `defmutation` sections, and uses the two-space `:cursive` argument style found in Fulcro's own source.
- `fulcro-new` now writes `tests.edn` and a sample Pathom resolver test. Previously Kaocha searched a nonexistent `test/` directory and `clj -M:test` passed without running any tests. The first-build check now runs `clj -M:test` too.
- `fulcro-new` no longer sets `:source-paths` in `shadow-cljs.edn`, which shadow-cljs ignores when `:deps` is set and reported with a warning on every build. Its clj-kondo configuration now accepts fulcro-spec's `=>` assertion arrows, and the root component no longer triggers an unused-binding warning.
- `fulcro-fix` no longer claims that dry4clj always exits 0. A failed scan now reports `ERROR`, and a new gotcha explains that dry4clj cannot read auto-resolved keywords such as `::pco/output`, so the advisory dry step reports `ERROR` on most Pathom code. Another new gotcha warns against running `cljfmt fix` over an `:indents`-only configuration.
- The project-workflows reference no longer cites a nonexistent `fulcro/checksum-state` helper for integration tests, and its `deps.edn` example now includes the `muuntaja` dependency its middleware example requires.

### Changed

- The skills now target Fulcro 3.10.0 and React 19. `fulcro-new` pins React 19.3.0, shadow-cljs 3.5.4, and refreshed versions of the other fixed dependencies, and fetches the latest `fulcro-inspect` alongside Fulcro, guardrails, and Pathom 3.
- `fulcro-new` now requires Java 21 or higher, as shadow-cljs 3.x does.
- `fulcro-upgrade` now also upgrades `com.fulcrologic/fulcro-inspect`, and it shows the release history and asks for confirmation on minor version jumps as well as major ones, since Fulcro 3.9 shipped breaking changes in a minor release. New gotchas cover the 3.8 Inspect migration, the 3.9 React 18 requirement and removed renderers, and the 3.10 docstring check.
- The `fulcro` skill documents the Fulcro 3.9 default React key for components with an ident, the `:use-hooks? :pure` option, the 3.10 compile error for a misplaced `defsc` docstring, the removed pre-React-18 renderers, and the `inspect.repl-tool` REPL flight recorder. The project-workflows reference adds Fulcro's headless JVM testing support.

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
