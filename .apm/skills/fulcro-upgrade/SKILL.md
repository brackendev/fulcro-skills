---
name: fulcro-upgrade
description: "Upgrade the Fulcro dependency in deps.edn to the latest released version and verify the project compiles and tests pass"
argument-hint: "[--report] [all]"
user-invocable: true
disable-model-invocation: true
---

# Upgrade Fulcro

Upgrade the `com.fulcrologic/fulcro` dependency in `deps.edn` to the latest released version on Clojars and verify the project still compiles and passes tests. Also upgrades `com.fulcrologic/fulcro-rad`, `com.fulcrologic/guardrails`, and `com.fulcrologic/fulcro-inspect` in the same pass when the project declares them. ClojureScript itself is upgraded by `/cljs-upgrade` in clojurescript-skills, not by this skill.

The argument grammar follows `CONVENTIONS.md` in the fulcro-skills source repository. The table below is complete for this skill.

## Arguments

| Input         | Target                                                                       |
|---------------|------------------------------------------------------------------------------|
| (no argument) | Update `:mvn/version` for `com.fulcrologic/fulcro` (and `fulcro-rad`, `guardrails`, `fulcro-inspect` if present) in `deps.edn`, then refresh dependencies, compile, and run tests |
| `all`         | Same as (no argument); accepted for family consistency                       |
| `--report`    | Print the current and latest released versions for each Fulcro artifact found in `deps.edn`. Surface the upstream release history for minor and major version jumps. No writes, no compile, no tests |

The skill rewrites version strings in a single `deps.edn`. `<path>` rows are not part of the standard scope vocabulary for this skill because the target is fixed.

## Mutation

Mutates `deps.edn` by default, replacing the `:mvn/version` value for `com.fulcrologic/fulcro`, and for `com.fulcrologic/fulcro-rad`, `com.fulcrologic/guardrails`, and `com.fulcrologic/fulcro-inspect` when present. Refreshes dependencies with `clj -P`, compiles the ClojureScript client with `npx shadow-cljs compile main`, runs the test suites, and refreshes clj-kondo imports. On compile or test failure, the skill reverts every version change in `deps.edn` to the recorded original values. With `--report`, the skill writes nothing and runs no compile or tests.

## Prerequisites

Verify `deps.edn` exists in the current directory and contains a `com.fulcrologic/fulcro` dependency. If not, stop and tell the user.

## Steps

Parse `$ARGUMENTS` to determine whether `--report` is present.

### 1. Record Current Versions

Read `deps.edn` and note the current `:mvn/version` value for each of:

- `com.fulcrologic/fulcro` (required)
- `com.fulcrologic/fulcro-rad` (if present)
- `com.fulcrologic/guardrails` (if present)
- `com.fulcrologic/fulcro-inspect` (if present, usually in the `:dev` alias)
- `com.wsscode/pathom3` (if present; report only, do not bump)

### 2. Fetch the Latest Released Versions

Query Clojars for each `com.fulcrologic/*` artifact found above:

```bash
curl -sSL -A 'Mozilla/5.0' \
  'https://clojars.org/api/artifacts/com.fulcrologic/fulcro' \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['latest_release'])"
```

Repeat for `fulcro-rad`, `guardrails`, and `fulcro-inspect` only if they are in `deps.edn`.

If a network call fails, ask the user for the target versions rather than guessing. Do not invent a version number.

If `--report` is present, print the previous and latest versions for each artifact found and stop without writing `deps.edn`, refreshing dependencies, or running the compile and tests:

```
Fulcro upgrade report:
  com.fulcrologic/fulcro:       <current> -> <latest>
  com.fulcrologic/fulcro-rad:   <current> -> <latest>   (or "not present")
  com.fulcrologic/guardrails:   <current> -> <latest>   (or "not present")
  com.fulcrologic/fulcro-inspect: <current> -> <latest> (or "not present")
  com.wsscode/pathom3:          <current> (informational only)
```

When the `com.fulcrologic/fulcro` minor or major version changes, also print the release history described in step 3 and the migration checks under Gotchas that apply to the range. Do not auto-apply or write.

### 3. Compare and Update

For each artifact, if the latest version matches the current version, report "Already at the latest version" and skip. Otherwise, update the `:mvn/version` value in `deps.edn` to the latest version. Preserve formatting and surrounding content.

Fulcro ships breaking changes in minor releases (3.9 removed renderers and required React 18 or later). The upstream `CHANGELOG.md` stops at 3.7.2 and is not maintained, so read the commit history between release tags instead. Fulcro tags releases as `fulcro-<version>`:

```bash
tmp="$(mktemp -d)"
git clone --quiet --filter=blob:none https://github.com/fulcrologic/fulcro.git "$tmp/fulcro"
git -C "$tmp/fulcro" log --no-merges --format='%cs %s' 'fulcro-<current>..fulcro-<latest>' \
  | grep -v 'maven-release-plugin'
```

If either tag is missing (for example, the project pins a snapshot), list the tags with `git -C "$tmp/fulcro" tag --sort=-creatordate` and ask the user which range to use.

When the minor or major version changes, show this history and the applicable Gotchas migration checks, and ask the user whether to proceed before writing the new version into `deps.edn`. Treat any commit subject that contains "BREAKING" as requiring explicit confirmation. Patch-level updates within the same minor version proceed without asking.

### 4. Refresh Dependencies

```bash
clj -P
```

If the dependency resolution fails, revert the `:mvn/version` in `deps.edn` to the original value, report the error, and stop.

### 5. Compile and Test

Run the ClojureScript build (development compile is enough as a sanity check; the `/fulcro-fix advanced` step covers the production build separately):

```bash
npx shadow-cljs compile main
```

Then run the test suites:

```bash
clj -M:test
npx shadow-cljs compile ci && npx karma start --single-run
```

If `karma` is not configured, fall back to `npx shadow-cljs compile test && node target/test.js`. If neither test runner is configured, skip the client test and record `test (client): SKIPPED`.

If compilation or any test suite fails, revert the version changes in `deps.edn`, restore the original state, report the error, and stop.

### 6. Refresh clj-kondo Imports

Fulcro's clj-kondo exports may have changed across the upgrade:

```bash
clj-kondo --copy-configs --dependencies --lint "$(clj -A:dev -Spath)"
```

### 7. Report

Print a summary:

```
Fulcro upgraded:
  com.fulcrologic/fulcro:       <old> -> <new>
  com.fulcrologic/fulcro-rad:   <old> -> <new>   (or "not present")
  com.fulcrologic/guardrails:   <old> -> <new>   (or "not present")
  com.fulcrologic/fulcro-inspect: <old> -> <new> (or "not present")
  com.wsscode/pathom3:          <unchanged>      (informational only)

  Compile:           PASS
  Tests (server):    PASS
  Tests (client):    PASS / SKIPPED
  clj-kondo imports: refreshed

Recommended next steps:
  /fulcro-fix advanced    # Production-build sanity check with externs inference
  Smoke test Fulcro Inspect in the browser
  Re-run any forms / load flows that exercise Fulcro internals
```

## Gotchas

- **Clojars `latest_release` includes snapshots when no stable release exists.** For long-lived stable channels this is fine; for new ecosystem packages the latest may be a snapshot. Confirm with the user before applying a snapshot version.
- **`com.wsscode/pathom3` is on an alpha track.** This skill reports the Pathom 3 version but does not bump it without explicit instruction; Pathom 3 alpha releases occasionally include breaking changes.
- **Minor Fulcro releases can break builds.** Read the tag-range commit history for every minor or major jump. The upstream `CHANGELOG.md` (the former `CHANGELOG.adoc` no longer exists) stops at 3.7.2.
- **Crossing 3.8 changes the Fulcro Inspect setup.** From 3.8 on, `com.fulcrologic.fulcro.inspect.preload` only logs "Inspect NOT installed". Search `shadow-cljs.edn` for it. The migration adds `com.fulcrologic/fulcro-inspect` to the development alias, replaces the preload with `com.fulcrologic.devtools.chrome-preload` (or `electron-preload`), calls `(ido (it/add-fulcro-inspect! app))` at client startup, and installs the new Chrome extension or Electron app from the fulcro-inspect releases page. The `fulcro` skill's Fulcro Inspect section shows the full setup. Report the needed change and ask before editing files other than `deps.edn`.
- **Crossing 3.9 requires React 18 or later and removes renderers.** Check `package.json` for `react` and `react-dom` 18 or later (Fulcro 3.10 is tested with React 19). Search the source for `multiple-roots-renderer`, `ident-optimized-render`, `keyframe-render2`, and `use-fulcro-mount`. Those namespaces and that function were removed, so any match fails to compile. Report the matches before applying the upgrade.
- **Crossing 3.10 rejects misplaced `defsc` docstrings.** A docstring placed after the argument list now fails compilation with "Docstring is in the wrong position". Move it before the argument list.
- **`shadow-cljs compile main` requires `node` and `npm` on `PATH`.** If either is missing, the compile step cannot run; report the missing prerequisite and stop.
- **Some projects pin Fulcro via a Git SHA** (`{:git/url "https://github.com/fulcrologic/fulcro" :sha "..."}`) rather than `:mvn/version`. This skill only handles the `:mvn/version` path; if a Git dependency is present, stop and tell the user.
- **Fulcro RAD couples loosely to Fulcro core but tightly to its own minor versions.** When bumping RAD across a major boundary, also check `com.fulcrologic/rad-*` companion libraries (`rad-datomic`, `rad-sql`, `rad-semantic-ui`) for compatible releases.
- **clj-kondo imports occasionally break across upgrades.** If the lint step floods after the refresh, delete `.clj-kondo/.cache/` and `.clj-kondo/imports/` and re-run `clj-kondo --copy-configs --dependencies`.
