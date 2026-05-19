---
name: fulcro-tidy
description: Tidy a Fulcro project (lint, format, test, advanced, dry); writes formatting by default
argument-hint: "[lint|format|test|advanced|dry] [--report] [all]"
allowed-tools: Bash, Read, Grep, Glob
user-invocable: true
disable-model-invocation: true
---

# Fulcro Tidy

Run lint, format, test, advanced-compilation, and duplicate-form checks on a Fulcro project. Fulcro projects are full-stack (Clojure server + ClojureScript client), so the `test` step aggregates server and client suites and the `advanced` step runs a shadow-cljs release build to surface production-only failures. The format step writes by default; the other four steps are pure-read of source.

See `CONVENTIONS.md` in the repo root for the argument grammar this skill follows.

## Arguments

| Input             | Target                                                                       |
|-------------------|------------------------------------------------------------------------------|
| (no argument)     | Run all five steps (`lint`, `format`, `test`, `advanced`, `dry`) on the project (`src`, `test`) |
| `all`             | Same as (no argument); accepted for family consistency                       |
| `lint`            | Run lint only                                                                |
| `format`          | Run format only                                                              |
| `test`            | Run server (Kaocha) and client (shadow-cljs / Karma) test suites             |
| `advanced`        | Run a shadow-cljs release build as a production-build sanity check           |
| `dry`             | Run the dry4clj duplicate-form scan only                                     |
| `<path>` `<glob>` | Restrict lint, format, and dry steps to those files or directories. The `test` and `advanced` steps ignore paths and always run the configured suites and builds |
| `--report`        | Replace `cljfmt fix` with non-writing `cljfmt check` in the format step      |

Step keywords are combinable (for example, `/fulcro-tidy lint test`). The `--report` flag may appear in any position. When `--report` is present without an explicit step keyword, every step still runs; only the format step's behavior changes.

## Mutation

Only the `format` step writes source. It runs `clj -M:cljfmt fix` by default, rewriting files in place. With `--report`, the step runs `clj -M:cljfmt check`, which exits non-zero when files would change but does not write. The `lint`, `test`, and `dry` steps are pure-read of source regardless of `--report`. The `advanced` step writes shadow-cljs release output under `resources/public/js/` (build artifact, not source) and is unaffected by `--report`.

## Steps

Parse `$ARGUMENTS` to determine which steps to run and whether `--report` is present. If no step keyword is supplied (or only `all` is supplied), run every step in order.

### 1. Lint

Bootstrap upstream Fulcro and Pathom clj-kondo exports if `.clj-kondo/.cache/imported` does not exist:

```bash
clj-kondo --copy-configs --dependencies --lint "$(clj -A:dev -Spath)"
```

Then lint the project sources:

```bash
clj-kondo --lint src test
```

If the project has no `test/` directory, lint `src` only.

Report pass if exit code is 0 and there are no errors. Show the clj-kondo output regardless.

Without the upstream Fulcro exports, the linter floods on `defsc`, `defmutation`, `defrouter`, and Pathom `defresolver` forms. Reach for the bootstrap step before reporting a flood as a genuine failure.

### 2. Format

Without `--report`, run cljfmt to fix formatting. `cljfmt` covers `.clj`, `.cljs`, and `.cljc`:

```bash
clj -M:cljfmt fix
```

With `--report`, run cljfmt in check mode (no writes):

```bash
clj -M:cljfmt check
```

Report pass if exit code is 0, fail otherwise. Show any formatting changes (under `fix`) or the diff (under `check`).

The project's `.cljfmt.edn` should include indentation entries for Fulcro macros (`defsc`, `defmutation`, `defrouter`, `defresolver`). The `/fulcro-new` skill writes them; older projects may need to be updated.

### 3. Test

Run the server test suite first, then the client test suite. Aggregate the exit codes.

#### Server tests (Kaocha)

```bash
clj -M:test
```

The `:test` alias should drive Kaocha against `src/test/**/*_test.clj`. Report pass if Kaocha exits 0 and the summary contains no failures or errors.

#### Client tests (shadow-cljs `:test` build)

If `shadow-cljs.edn` defines a `:test` build, compile it and run the resulting test page in headless Chrome via Karma:

```bash
npx shadow-cljs compile ci && npx karma start --single-run
```

Alternatively, when a Node-friendly test entry exists:

```bash
npx shadow-cljs compile test && node target/test.js
```

Detect the test target from `shadow-cljs.edn`. If neither path is configured, skip the client test step and record `test (client): SKIPPED`.

Report `test: PASS` only when both server and client suites pass. Report `test: FAIL` if either fails, and surface the failing summary.

### 4. Advanced

Run a shadow-cljs release build as a production-build sanity check:

```bash
npx shadow-cljs release main
```

This compiles under `:advanced` optimizations, runs externs inference, and surfaces warnings that do not appear in development builds. Report fail if exit code is non-zero, or if the output contains `WARNING:` lines from the Google Closure Compiler about property renaming or missing externs.

If the project does not define a `:main` (or equivalent) build, ask the user for the build name. Do not guess.

The advanced step is slow on first run (whole-program optimization). shadow-cljs caches across runs.

### 5. Dry

Scan for duplicate top-level forms with [dry4clj](https://github.com/unclebob/dry4clj):

```bash
clj -M:dry4clj src test
```

If the project has no `test/` directory, scan `src` only.

dry4clj exits 0 whether or not it finds candidates, so the step must inspect output. Report pass only when exit code is 0 and stdout contains the literal `No duplicate candidates found.`. Otherwise report fail and show the reported candidates. The project's `deps.edn` must define a `:dry4clj` alias; if the alias is missing, the Clojure CLI exits non-zero and the step fails.

Upstream dry4clj scans `.clj`, `.cljc`, and `.cljs` files by default; the Clojure server, the cljc shared layer, and the ClojureScript client are all covered.

## Report

After running all requested steps, print a summary:

```
Fulcro Tidy Results:
  Lint:     PASS/FAIL/SKIPPED
  Format:   PASS/FAIL/SKIPPED
  Test:     PASS/FAIL/SKIPPED  (server: PASS, client: PASS)
  Advanced: PASS/FAIL/SKIPPED
  Dry:      PASS/FAIL/SKIPPED
```

If `--report` was passed, append `(report mode: format checked, not written)` after the summary. If any step fails, stop and report the failure. Do not continue to subsequent steps.

## Gotchas

- **The lint step must bootstrap Fulcro hooks first.** Without `clj-kondo --copy-configs --dependencies`, `defsc` and `defmutation` forms generate hundreds of false positives. Re-run the bootstrap after every Fulcro upgrade.
- **`shadow-cljs release main` requires `node` and `npm` on `PATH`.** If either is missing, the advanced step is reported as SKIPPED rather than FAIL.
- **Karma requires Chrome on `PATH`.** `karma-chrome-launcher` looks for a `chrome` (or `chromium`) binary. On headless CI, install the Chrome runtime first (for example, `apt-get install -y chromium-browser` on Debian-based images).
- **shadow-cljs holds a daemon process.** `shadow-cljs watch` from another terminal can lock the `:main` build slot. Stop it (`npx shadow-cljs stop`) before running `release main`.
- **The dry step skips `.cljd` files.** Fulcro projects do not use ClojureDart, so this is irrelevant here, but the same `:dry4clj` alias copied from a ClojureDart project will not pick up `.cljd` either without the brackendev/dry4clj `add-cljd-extension` branch.
- **Server tests via Kaocha and client tests via shadow-cljs run in different JVMs.** A failing test in one suite does not abort the other. The aggregate report distinguishes server (PASS) from client (PASS) so partial failures are clear.
- **Pathom resolvers can fail at test time without failing at compile time.** Invalid `::pco/output` declarations (for example, declaring an attribute the resolver does not actually return) only surface during query execution. The test suite is the place to catch this.
- **`dry` is a duplicate-form scan, not a dry-run mode.** The step keyword names the dry4clj tool. To preview the format step without writing, pass `--report`, not `dry`.
