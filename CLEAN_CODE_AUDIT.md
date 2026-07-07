# Clean Code Audit

**Date**: 2026-06-30
**Scope**: Full audit (subset: top-value 10 files by size + churn — `cli.ts`, `evaluator.ts`, `installer.ts`, `formatter.ts`, `security-auditor.ts`, `library.ts`, `updater.ts`, `uninstaller.ts`, `config.ts`, `ingester.ts`)
**Files Audited**: 10 source files (~14.5K LOC) of 58 total (~24.7K LOC)
**Primary language**: TypeScript (Node CLI / Ink TUI)
**Source standard**: bbv Clean Code Cheat Sheet V2.2 (Urs Enzler)

## Summary

| Severity | Count |
| -------- | ----- |
| Critical | 8     |
| Major    | 19    |
| Minor    | 9     |
| Info     | 3     |

> **Headline:** Three of the eight Criticals are **real security/correctness bugs in `security-auditor.ts`** — the module whose whole job is to flag dangerous skills. They are the most valuable output of this audit and should be fixed first. The remaining Criticals are a recurring **swallowed-exception** pattern that hides data loss, and the **God-file `cli.ts`** (6,618 lines).

---

## Findings

### Critical

#### [Be precise / correctness]: Security allowlist bypassed by subdomain confusion

**File**: `src/security-auditor.ts:80`
**Principle**: Be precise (Design) — a security check that is imprecise is worse than none.

The external-URL exfiltration check uses `https?:\/\/(?!github\.com|localhost|127\.0\.0\.1|example\.com)`. The negative lookahead is unanchored, so `https://github.com.evil.com/payload` begins with `github.com`, the lookahead fails, and the **malicious URL is never flagged**. Verified by direct read.

**Fix direction**: Anchor the host with a terminator — `(?!(?:github\.com|localhost|127\.0\.0\.1|example\.com)(?:[:/]|$))` — or parse the URL and compare `hostname` exactly against an allowlist.

#### [Don't swallow / Fail fast]: Truncated match re-test silently drops permissions

**File**: `src/security-auditor.ts:398`
**Principle**: Be precise — detection and reporting must use the same data.

`scanCode` stores each match truncated to `trimmed.slice(0, 120) + "..."` (lines 352–353). `analyzePermissions` then re-derives the permission type by re-running `pattern.pattern.test(match.match)` against that **truncated** string (line 402). If a risky token (`exec(`, `spawn(`) sits past column 120 — common on long/minified lines — the original line matched but the re-test fails, so the permission is silently dropped from the verdict. Verified by direct read.

**Fix direction**: Carry the matched `permissionType` through from `scanCode` into `CodeScanMatch`; delete the second re-testing loop entirely.

#### [Be precise / DRY single source of truth]: Verdict counts diverge from displayed report

**File**: `src/security-auditor.ts:468`
**Principle**: Be precise; single source of truth.

`calculateVerdict` counts `criticalCount`/`warningCount` over the **raw, non-deduplicated** `scanResults`, while the report shows **deduplicated** matches (`deduplicateMatches`, line 671). One line like `const {exec, spawn} = require('child_process')` produces ≥2 critical matches at the same `file:line`, inflating `criticalCount` toward the `>= 10` "dangerous" threshold. The verdict is driven by counts the user never sees.

**Fix direction**: Deduplicate by `file:line` before counting, reusing the same `deduplicateMatches` helper so verdict and report agree.

#### [Don't swallow exceptions]: Corrupted-lock catch-all wipes the installed-skill registry

**File**: `src/library.ts:114`
**Principle**: Don't swallow exceptions (Critical) — leaves the system inconsistent.

A bare `catch {}` treats _every_ post-`readFile` failure as "corrupted lock," backs it up, and returns an empty lock — silently discarding the user's entire installed-skill registry. A transient or unexpected error is indistinguishable from real corruption, and the original error is discarded.

**Fix direction**: Catch only `SyntaxError`/validation errors; let unexpected types propagate; log the actual error so "corruption" is diagnosable.

#### [Don't swallow exceptions]: Rollback failure during update is silently swallowed

**File**: `src/updater.ts:523`
**Principle**: Don't swallow exceptions (Critical).

In `updateSkill`'s rollback path, an inner `catch {}` swallows failures while restoring the user's original skill from the `.bak` directory. If rollback fails, the user is left with a missing/half-written skill and **no diagnostic at all**.

**Fix direction**: At minimum `debug()`/warn the rollback failure and surface it in the returned reason so a failed restore is observable.

#### [Don't swallow exceptions]: Filesystem-removal failures reported as success

**File**: `src/uninstaller.ts:465`
**Principle**: Don't swallow exceptions (Critical) — destructive op silently partial.

Directory/symlink removal failures (and `removeAgentsMdBlock` failures at line 494) are caught, appended to a log array, and execution continues; `executeRemoval` returns normally. A failed `rm` (EACCES, EBUSY) is invisible to the caller's exit code, so a partial uninstall reports success.

**Fix direction**: Track removal failures and throw an aggregated error (mirroring the relocation path at 398–402), or return a `{log, failures}` result the caller must inspect for a non-zero exit.

#### [Single Responsibility Principle / God file]: cli.ts is a 6,618-line module

**File**: `src/cli.ts:1`
**Principle**: SRP; Classes/modules should be small.

A single module owns arg parsing, 26 help-text printers, 30+ command handlers, output formatting, stdin reading, and process lifecycle — one reason to change for every command. `cmdInstall` alone is **~916 lines** (2738–3654) and `cmdBundle` **~700 lines** (5374–6075). This drives the testability problem below: `cli.test.ts` cannot unit-test the handlers and instead shells out to a subprocess (see Major: testability).

**Fix direction**: Split into `src/cli/parse-args.ts`, `src/cli/help/*`, and one module per command (`src/cli/commands/install.ts`, `bundle.ts`, …) behind a command registry; keep `cli.ts` as a thin dispatcher.

#### [Use exceptions, not exit codes / Testability]: 133 process.exit() calls make handlers untestable

**File**: `src/cli.ts:2754`
**Principle**: Use exceptions, not return codes (Major→Critical here for testability); Design for testability.

Command handlers call `process.exit()` 133 times to signal errors, terminating the process rather than returning/throwing. Consequence (confirmed): `cli.test.ts` imports only `parseArgs`/`isCLIMode` and runs every command through a **subprocess** wrapper — the 30+ handlers (`cmdInstall`, `cmdBundle`, …) have **no in-process unit tests**, only slow integration tests (an inverted test pyramid).

**Fix direction**: Have handlers `throw` a typed `CliError{exitCode, message}`; centralize exit in `runCLI`'s catch. Handlers become pure and unit-testable.

### Major

#### [SRP / Long Method]: God functions clustered around clone → audit → atomic-swap

**File**: `src/library.ts:428` (also `src/updater.ts:343`, `src/uninstaller.ts:319`, `src/ingester.ts:51`, `src/installer.ts:611`)
**Principle**: SRP; Long Method (>20 lines, nesting >3). _(Cross-file pattern — enumerated here once, not double-counted per file.)_

The same shape recurs across the lifecycle modules: a single ~180–214-line function does validation + clone + audit/verify + path-containment + atomic swap + rollback + lock write. Instances: `updateLibrarySkill` (~207 lines), `updateSkill` (~214), `executeRemoval` (~180, nesting 5+), `ingestRepo` (~190), `installer.executeInstall` (~60, six responsibilities).

**Fix direction**: Establish a shared decomposition vocabulary — `cloneToTemp()`, `verify()`, `validateContainment()`, `atomicSwap(src, target)`, `buildLockEntry()` — and have each top-level function orchestrate ≤80 lines.

#### [Don't swallow exceptions]: Corrupted-config catch-all conflates causes

**File**: `src/config.ts:296`
**Principle**: Don't swallow exceptions; catch specific exceptions.

A bare `catch {}` treats every failure after `readFile` (including `copyFile`/`saveConfig` errors inside the block) as "corrupted config," discarding the original error and masking the real cause (disk full, permission denied). Major rather than Critical because the default config is regenerable.

**Fix direction**: Scope the `try` to `JSON.parse`, bind the error, and let write failures propagate distinctly from corruption.

#### [Temporal coupling / side-effecting input]: mergeWithDefaults mutates caller's parsed config

**File**: `src/config.ts:257`
**Principle**: No out/ref arguments; avoid hidden side effects.

`mergeWithDefaults` mutates the caller's parsed array in place via `providers.splice(...)` and returns the same reference, silently altering the `JSON.parse` result and breaking referential transparency.

**Fix direction**: Clone first (`(config.providers ?? []).map(p => ({...p}))`) and splice into the copy.

#### [Don't swallow exceptions]: Containment realpath catch coerced to false

**File**: `src/library.ts:246`
**Principle**: Catch where you can react meaningfully.

`libraryPathRealpathIsContained` wraps both `realpath` calls in `catch { return false }`. An `EACCES` (or any non-`ENOENT`) failure becomes "not contained," surfacing a misleading "escapes library skills directory" rejection instead of the true I/O error.

**Fix direction**: Treat only `ENOENT` as the false case; propagate other error codes.

#### [Control-flow-by-exception]: Conflict detection branches on error-message substrings

**File**: `src/installer.ts:909`
**Principle**: Don't use exceptions for control flow; be precise.

`checkConflict` relies on `access()` throwing to detect non-existence, then disambiguates its own re-thrown error by string-matching `err.message?.includes("--force")`. `isAuthError` (line 335) similarly drives the HTTPS→SSH fallback off locale-dependent git stderr substrings (`"authentication failed"`).

**Fix direction**: Use non-throwing existence checks and stable signals (exit codes / typed sentinel errors); treat substring lists as a tested best-effort heuristic.

#### [SRP / God function]: formatSecurityReport is a ~240-line render function

**File**: `src/security-auditor.ts:712`
**Principle**: SRP; Long Method.

`formatSecurityReport` (712–953) handles header box, source line, threat summary, per-category findings, file-group truncation, and footer in one function with deep nesting and inline layout math.

**Fix direction**: Extract `renderHeaderBox`, `renderSourceLine`, `renderThreatSummary`, `renderFindings`, `renderFooter`, each returning `string[]`.

#### [DRY]: Duplicated single-source-of-truth maps in security-auditor

**File**: `src/security-auditor.ts:596`
**Principle**: DRY; single source of truth.

`CATEGORY_TO_PERM` (596) duplicates the category→permission mapping already encoded by `permissionType` in `SCAN_PATTERNS` and can drift (it even invents `'credentials'`/`'obfuscation'` values that aren't valid `PermissionRequest['type']`). `severityOrder` is also defined twice (line 375 in `scanCode` and module-level `SEVERITY_ORDER` at 607). Category `description` strings are copy-pasted 6–7× across `SCAN_PATTERNS`.

**Fix direction**: Derive the permission label from `SCAN_PATTERNS.permissionType`; keep one `SEVERITY_ORDER`; attach one `{name, description}` per category and list patterns under it.

#### [SRP / DRY]: formatter mixes I/O into a pure presentation module and duplicates table rendering

**File**: `src/formatter.ts:761`
**Principle**: SRP; DRY; KISS.

`formatSkillDetail` (761) and `formatSkillInspect` (913) call `await countFiles(skill.path)` — filesystem I/O in a render module, forcing both to be async and non-deterministic. Separately, `formatSearchResults` (595) is a near-verbatim copy of `formatGroupedTable`'s column-width + row-render logic, and the eval-summary block (786–834) is duplicated in `formatSkillInspect` (928–963).

**Fix direction**: Require callers to pre-populate `skill.fileCount`; extract a shared table renderer and `renderEvalSummary(summaries, indent)`.

#### [Long Method / God function]: formatSkillInspect / formatSkillDetail

**File**: `src/formatter.ts:882`
**Principle**: Long Method.

`formatSkillInspect` (~140 lines, 882–1022) and `formatSkillDetail` (~110 lines, 738–847) each inline header, shared-info, eval summary, description wrap, allowed-tools, and installations.

**Fix direction**: Decompose into `renderHeader`/`renderSharedInfo`/`renderInstallations`/`renderWarnings` helpers shared between the two.

#### [SRP / temporal coupling]: installer.resolveProvider mixes resolution, UI, and persistence

**File**: `src/installer.ts:779`
**Principle**: SRP; Decouple construction from runtime; make logical dependencies physical.

`resolveProvider` (~88 lines) mixes config filtering, multiple error paths, the interactive checkbox picker, persistence, and a lazy `await import("./config")` (855) buried mid-function — a hidden dependency that defeats testability.

**Fix direction**: Split non-interactive resolution from the interactive picker; hoist the config import to module scope or inject the persistence function.

#### [Inconsistent state / no rollback]: Partial multi-provider install leaves linked dirs

**File**: `src/installer.ts:682`
**Principle**: Be precise; keep a running (consistent) system.

`executeInstallAllProviders` installs the primary then loops creating symlinks. If `symlink`/`mkdir` throws partway, earlier providers stay linked but the call rejects — a partially-linked, inconsistent set with no cleanup.

**Fix direction**: Collect per-provider results; roll back created links on failure, or aggregate failures and report without aborting.

#### [Long Parameter List / Flag arguments]: positional booleans and mode flags

**File**: `src/installer.ts:868` (also `src/uninstaller.ts:319`, `src/library.ts:760`, `src/cli.ts:242`)
**Principle**: Few arguments (>3 smell); no selector/flag arguments. _(Cross-file pattern.)_

`buildInstallPlan` takes 7 positional params incl. a boolean `force` and `scope` (`(source, tempDir, sourceDir, name, provider, false, "global")` is unreadable). `executeRemoval(plan, symlinkTo?, relocation?)` encodes mutually-exclusive modes as two optional positionals. `activate/installLibrarySkill` branch on a `force` flag. `ParsedArgs.flags` (cli.ts:242) is a 40-field bag every command receives whole (ISP/too-much-information).

**Fix direction**: Replace positional booleans with an options object; model modes as a discriminated union (`{mode:'relocate'|'symlink'|'plain'}`); group cli flags into per-command sub-objects.

#### [DRY]: Error-message coalescing duplicated across modules

**File**: `src/library.ts:391` (also `src/ingester.ts:57`, `src/installer.ts:331`, `src/security-auditor.ts:321`)
**Principle**: DRY; avoid `any` on the error path. _(Cross-file pattern.)_

`err?.stderr || err?.message || String(err)` and `err?.message ?? String(err)` recur 7+ times in `library.ts` alone, and `catch (err: any)` + `err.message` recurs across `ingester`, `installer`, `security-auditor` — `any` defeats type-safety and an unexpected non-Error throw yields `undefined`.

**Fix direction**: Extract one `toErrorMessage(err: unknown): string` helper; use `catch (err: unknown)` with `err instanceof Error ? … : String(err)`.

#### [DRY]: Duplicated skill-construction and path-resolution in installer

**File**: `src/installer.ts:468`
**Principle**: DRY.

The frontmatter→`DiscoveredSkill` mapping is copy-pasted for the root skill (468–479) and nested skills (510–521), with a third partial copy in `validateSkill` (449). `parseSource` (115) is a ~95-line nested ref/subpath/colon parser.

**Fix direction**: Extract `frontmatterToSkill(content, relPath, fallbackName)`; extract a `parseOwnerRepoRefSubpath` tokenizer.

#### [DRY / God function]: evaluator scorers repeat boilerplate; buildFixPlan does too much

**File**: `src/evaluator.ts:329`
**Principle**: DRY; Long Method.

The seven `scoreXxx` functions repeat identical plumbing (findings/suggestions arrays, score accumulator, `containsAny` bucketing, `Math.min(10, Math.round(score))` clamp, identical `CategoryResult` shape). `buildFixPlan` (1197) is ~120 lines applying eight distinct fixes by sequential mutation of `working`/`fmStr`.

**Fix direction**: Table-driven scorer where each category declares only its rules; split `buildFixPlan` into one function per fix iterated over a list.

#### [Risky logic / weak tests]: naive unifiedDiff produces misleading hunks

**File**: `src/evaluator.ts:1347`
**Principle**: Be precise; POUTing (add coverage for boundary cases).

`unifiedDiff` is a self-described "naive" non-LCS diff emitting all removals then all additions; on interleaved changes it yields an invalid-looking hunk, yet tests cover only single-line and empty cases.

**Fix direction**: Use a real diff algorithm, or constrain+document it as preview-only and add multi-region/adjacent-edit tests.

#### [Test pyramid / Hidden test smells]: integration tests masquerade as unit tests

**File**: `src/library.test.ts:997` (also `src/installer.test.ts:472`, `src/cli.test.ts:39`)
**Principle**: Unit tests must be Fast & Isolated; Test pyramid (avoid inverted). _(Cross-file pattern.)_

`library.test.ts` shells out to the real `git` binary (init/config/commit/clone); `installer.test.ts` invokes real `git`/`npx` ("relies on CI and dev machines") and mutates global cwd via `process.chdir`; `cli.test.ts` runs every command via a **subprocess** wrapper because the handlers `process.exit`. These are slow, environment-dependent, order-sensitive — an inverted pyramid.

**Fix direction**: Inject the exec/clone runner as a seam and mock it; make `isExistingLocalDir` take a base-dir param instead of `chdir`; unit-test cli handlers once they throw instead of exit.

#### [Test smell]: tautological / literal-only assertions give false coverage

**File**: `src/updater.test.ts:471` (also `src/evaluator.test.ts:141`)
**Principle**: A test that doesn't test anything (Critical smell) / over-broad assertions.

`updater.test.ts` "up-to-date / outdated" tests assert `manifestCommit === installedCommit` on literals and **never call `checkOutdated`**; `evaluator.test.ts` scoring tests assert only loose bands (`overallScore > 70`, `grade != 'F'`), so rubric regressions inside the band go undetected.

**Fix direction**: Replace literal-only tests with ones that invoke the SUT (the injectable-override `checkOutdated` tests already exist); add exact-score assertions per scorer.

#### [SRP / God function]: ingestRepo runs the whole ingest pipeline inline

**File**: `src/ingester.ts:51`
**Principle**: SRP; Long Method. _(Listed under the cross-file God-function pattern above; kept here for its distinct best-effort-swallow concern.)_

`ingestRepo` (~190 lines) parses, clones, dedupes, runs two eval pipelines, assembles the index, merges bundles, and writes — with three best-effort `try/catch` blocks (102, 158, 190) that swallow eval/read failures into `debug()` only, so a systematically broken evaluator silently produces an index with no eval data and no visible warning.

**Fix direction**: Extract `enrichSkill()`, `runStaticEval()`, `runProviderEvals()`; accumulate failures into `IngestResult.warnings` so CI can detect silent eval breakage.

### Minor

#### [Avoid globals / artificial coupling]: \_\_CLI_NO_COLOR untyped global spans three modules

**File**: `src/cli.ts:6436` (also `src/formatter.ts:9`, `src/logger.ts:18`)
**Principle**: No Singletons/global state; make logical dependencies physical. _(Cross-file pattern.)_

Color disabling is a hidden `(globalThis as any).__CLI_NO_COLOR` written in `cli.ts` and read in `formatter.ts` and `logger.ts` via `as any` casts — an implicit, untyped dependency read on every render call.

**Fix direction**: Thread a typed `NoColor` flag through an output context, or a typed module setter mirroring `setVerbose`.

#### [Magic numbers]: timeouts, short-hash, and SHA regex scattered everywhere

**File**: `src/cli.ts:1311` (also `installer.ts:362`, `library.ts:305`, `updater.ts:389`, `security-auditor.ts:353`, `formatter.ts:527`, `config.ts:166`)
**Principle**: Magic numbers / no named constants. _(Cross-file pattern.)_

`30_000`/`60_000`/`120_000` timeouts, `slice(0,7)` short-hash, `/^[0-9a-f]{40}$/` SHA regex, `>= 10`/`< 3` security thresholds, and formatter widths (`15`, `28`, `76`) appear as bare literals across all audited files.

**Fix direction**: Hoist named constants (`CLONE_TIMEOUT_MS`, `SHORT_HASH_LEN`, `GIT_SHA1_RE`, `DANGEROUS_CRITICAL_THRESHOLD`, `CREATOR_MAX_WIDTH`, …) per module or in a shared `constants.ts`.

#### [Dead code]: unused imports, fields, and helpers

**File**: `src/evaluator.ts:38` (also `formatter.ts:25`, `installer.ts:976`, `evaluator.ts:1310`, `evaluator.ts:2026`)
**Principle**: Dead code / clutter / YAGNI.

Unused `access` import (evaluator:38); unused `ansi.bg*` helpers (formatter:25); always-`false` unused `isLocalSource` field (installer:976); a no-op `if` whose body is only a comment (evaluator:1310); a `void widthHint` "reserved for future" param (evaluator:2026).

**Fix direction**: Delete each; version control remembers.

#### [Naming / Negative conditional]: double-negative guards and misleading names

**File**: `src/installer.ts:81` (also `library.ts:431`, `evaluator.ts:1455`)
**Principle**: Positive conditionals; descriptive names.

`isExistingLocalDir` opens with `if (!input.includes("/") && !input.includes("\\"))` (double negative). `updateLibrarySkill`'s `_overrides` param (underscore implies unused) is actively used as a test seam. `detectGitAuthor`'s name/comment say "git config user.name" but it runs `--global` only.

**Fix direction**: Introduce `const hasSeparator = …; if (!hasSeparator) return false;`; rename `_overrides`→`overrides`; drop `--global` or document the scope.

#### [DRY]: index filename & marker conventions rebuilt in multiple functions

**File**: `src/ingester.ts:227` (also `uninstaller.ts:198`)
**Principle**: DRY.

The `${owner}_${repo}.json` index name is built in `ingestRepo` (227) and re-built in `removeRepoIndex` (269); the AGENTS.md marker prefix list `["agent-skill-manager","skill-manager","pskills"]` is duplicated in `uninstaller` (198 and 523).

**Fix direction**: Extract `indexFileName(owner, repo)` and module-level `MARKER_PREFIXES` + `buildMarker(prefix, name)`.

#### [Primitive obsession]: provider+scope identity encoded as `::`-joined string

**File**: `src/uninstaller.ts:99`
**Principle**: Prefer value objects to primitives.

Provider+scope identity is built as `${provider}::${scope}` into a Set then decoded with `.split("::")[0]` (115). A provider name containing `::` corrupts the key.

**Fix direction**: Use a typed `{provider, scope}` key or a `Map` with a stable key helper.

#### [Magic strings]: em-dash sentinel and `.DS_Store` literals

**File**: `src/formatter.ts:121` (also `uninstaller.ts:306`)
**Principle**: Magic strings; encapsulate boundary conditions.

The `"—"` "missing value" placeholder is sprinkled across ~10 sites in `formatter.ts`; `.DS_Store` (and the gitkeep policy) is an inline literal in `cleanEmptyParentDirs`.

**Fix direction**: `const EMPTY_CELL = "—"`; `const IGNORED_DIR_ENTRIES = [".DS_Store"]`.

#### [Conditional to encapsulate]: nested ternaries for path resolution

**File**: `src/updater.ts:473`
**Principle**: Encapsulate conditionals.

`configuredPath` nests a ternary inside a ternary plus a hardcoded `~/.${provider}/skills` fallback.

**Fix direction**: Extract `resolveConfiguredPath(providerConfig, scope, provider)` with early returns.

#### [Dead code / tech debt]: shipped TODO documenting a redundant network round-trip

**File**: `src/updater.ts:583`
**Principle**: Inappropriate information (belongs in a tracker).

A multi-line TODO documents a known redundant clone/refetch left unaddressed in shipped code.

**Fix direction**: Implement the `knownLatestCommit` optimization or move the note to an issue.

### Info

#### [Primitive obsession]: dotted pseudo-keys stand in for nested frontmatter

**File**: `src/evaluator.ts:79`
Frontmatter is typed `Record<string,string>` with magic dotted keys like `'metadata.author'`. _Fix:_ a typed `Frontmatter` model.

#### [Data-as-code]: 138-line DEFAULT_PROVIDERS literal embedded in source

**File**: `src/config.ts:25`
Provider priority is encoded only by array position + comments. _Fix:_ a data file or explicit `priority` field.

#### [Testability seam leaks into production API]: `_overrides` params in public signatures

**File**: `src/updater.ts:330` (also `library.ts:431`)
`_UpdateTestOverrides` etc. bake test wiring into public call signatures. _Fix:_ inject via a constructed object/closure rather than underscore-prefixed params.

---

## Implementation Plan

### Phase 1 — Critical (do first)

- [ ] **1.1 — Fix the external-URL allowlist bypass**
  - File: `src/security-auditor.ts:80`
  - Principle: Be precise / correctness
  - Effort: ~30m
  - Depends on: none
  - Acceptance: a test asserts `https://github.com.evil.com/x` IS flagged (network) and `https://github.com/org/repo` is NOT.

- [ ] **1.2 — Stop dropping permissions on truncated matches**
  - File: `src/security-auditor.ts:352,398`
  - Principle: Be precise / Don't lose information
  - Effort: ~1.5h
  - Depends on: none
  - Acceptance: a long line (>120 cols) with `exec(` past column 120 yields an `exec`/process permission in the verdict; the re-testing loop at 398 is removed.

- [ ] **1.3 — Make the verdict count deduplicated matches**
  - File: `src/security-auditor.ts:468`
  - Principle: Single source of truth
  - Effort: ~1h
  - Depends on: none
  - Acceptance: a line matching two critical patterns at one `file:line` counts as 1; verdict and displayed report agree under a test.

- [ ] **1.4 — Add security-detection tests for the untested critical patterns**
  - File: `src/security-auditor.test.ts` (patterns at auditor `rm -rf`/`spawn`/`Bun.spawn`/`-c`/`new Function`/secret-key)
  - Principle: Risky-untested (TDD)
  - Effort: ~2h
  - Depends on: 1.1–1.3 (so new tests assert corrected behavior)
  - Acceptance: each critical pattern has a positive + near-miss negative test; `analyzeSource` fetch failure is mocked and asserted not to bias toward "safe".

- [ ] **1.5 — Replace data-loss `catch {}` blocks with scoped handling**
  - File: `src/library.ts:114`, `src/updater.ts:523`, `src/uninstaller.ts:465,494`
  - Principle: Don't swallow exceptions
  - Effort: ~3h
  - Depends on: none
  - Acceptance: lock-load catches only parse/validation errors (others propagate); rollback + fs-removal failures surface in the return value/exit code with a regression test for each.

- [ ] **1.6 — Make cli command handlers throw instead of `process.exit`**
  - File: `src/cli.ts` (133 sites; introduce `CliError` + central handler in `runCLI`)
  - Principle: Use exceptions, not exit codes; Design for testability
  - Effort: ~1d
  - Depends on: none (enabling refactor for 1.7 and 2.1)
  - Acceptance: `cli.test.ts` unit-tests at least 3 handlers in-process (no subprocess); `runCLI` maps `CliError.exitCode` to `process.exit` in one place.

- [ ] **1.7 — Carve cli.ts into a dispatcher + per-command modules**
  - File: `src/cli.ts:1` → `src/cli/parse-args.ts`, `src/cli/help/*`, `src/cli/commands/*`
  - Principle: SRP / God file
  - Effort: ~2d
  - Depends on: 1.6
  - Acceptance: `cli.ts` ≤300 lines (dispatcher only); each command in its own ≤200-line module; build + existing e2e green.

### Phase 2 — Major

- [ ] **2.1 — Decompose the clone→audit→swap God functions**
  - File: `src/library.ts:428`, `src/updater.ts:343`, `src/uninstaller.ts:319`, `src/ingester.ts:51`, `src/installer.ts:611`
  - Principle: SRP / Long Method
  - Effort: ~2d
  - Depends on: 1.5 (error handling settles first)
  - Acceptance: each top-level function ≤80 lines orchestrating named helpers; shared `atomicSwap`/`verify` helpers; tests green.

- [ ] **2.2 — Extract a shared `toErrorMessage(err: unknown)` and drop `any` on error paths**
  - File: `src/library.ts`, `src/ingester.ts`, `src/installer.ts`, `src/security-auditor.ts`
  - Principle: DRY; type safety
  - Effort: ~2h
  - Depends on: none
  - Acceptance: one helper; no `catch (err: any)` in audited files; behavior unchanged under tests.

- [ ] **2.3 — Replace positional/flag arguments with options objects & discriminated unions**
  - File: `src/installer.ts:868`, `src/uninstaller.ts:319`, `src/library.ts:760`, `src/cli.ts:242`
  - Principle: Few arguments; no flag/selector args
  - Effort: ~1d
  - Depends on: 2.1
  - Acceptance: no >3-param call sites in audited files; `executeRemoval` takes a mode union; `ParsedArgs.flags` grouped per command.

- [ ] **2.4 — De-duplicate formatter table & eval-summary rendering; remove I/O from formatters**
  - File: `src/formatter.ts:595,761,786,882`
  - Principle: DRY; SRP
  - Effort: ~1d
  - Depends on: none
  - Acceptance: one table renderer + one `renderEvalSummary`; formatters are synchronous/pure (callers pass `fileCount`); snapshot tests unchanged.

- [ ] **2.5 — Consolidate security-auditor maps and split formatSecurityReport**
  - File: `src/security-auditor.ts:596,607,712`
  - Principle: DRY; SRP
  - Effort: ~1d
  - Depends on: 1.1–1.3
  - Acceptance: single `SEVERITY_ORDER`; permission label derived from `SCAN_PATTERNS`; `formatSecurityReport` split into ≤5 zone renderers.

- [ ] **2.6 — Fix remaining swallowed/control-flow exception paths**
  - File: `src/config.ts:296`, `src/library.ts:246`, `src/installer.ts:909,335`, `src/installer.ts:682`
  - Principle: Catch specific exceptions; don't use exceptions for control flow; consistent state
  - Effort: ~1d
  - Depends on: 2.2
  - Acceptance: `checkConflict`/`isAuthError` branch on booleans/exit codes not message substrings; multi-provider install rolls back or aggregates failures; config catch scoped to parse.

- [ ] **2.7 — Stop mutating caller input in mergeWithDefaults**
  - File: `src/config.ts:257`
  - Principle: No out/ref arguments
  - Effort: ~30m
  - Depends on: none
  - Acceptance: input `config.providers` is unchanged after the call (test); output is a new array.

- [ ] **2.8 — Table-drive evaluator scorers; split buildFixPlan; harden unifiedDiff**
  - File: `src/evaluator.ts:329,1197,1347`
  - Principle: DRY; Long Method; Be precise
  - Effort: ~1.5d
  - Depends on: none
  - Acceptance: scorers share one plumbing helper; `buildFixPlan` is a list of single-fix functions; `unifiedDiff` either uses a real algorithm or is documented preview-only with multi-region tests.

- [ ] **2.9 — Convert integration-as-unit tests into seamed unit tests**
  - File: `src/library.test.ts:997`, `src/installer.test.ts:472,1486`, `src/cli.test.ts`
  - Principle: Fast & Isolated tests; test pyramid
  - Effort: ~1.5d
  - Depends on: 1.6, 2.1 (seams exist)
  - Acceptance: git/npx/clone runners are injected & mocked; no `process.chdir` in tests; cli handlers tested in-process; suite faster and host-independent.

- [ ] **2.10 — Replace tautological/literal-only tests with SUT-invoking tests**
  - File: `src/updater.test.ts:471`, `src/evaluator.test.ts:141`
  - Principle: A test that doesn't test anything
  - Effort: ~3h
  - Depends on: none
  - Acceptance: outdated/score tests call `checkOutdated`/scorers; at least a few exact-score assertions lock the rubric.

### Phase 3 — Minor

- [ ] **3.1 — Introduce a typed no-color flag, remove the `__CLI_NO_COLOR` global**
  - File: `src/cli.ts:6436`, `src/formatter.ts:9`, `src/logger.ts:18`
  - Principle: No global state; make dependencies physical
  - Effort: ~2h
  - Depends on: 1.7
  - Acceptance: no `(globalThis as any).__CLI_NO_COLOR`; color flag threaded/typed; no-color e2e still works.

- [ ] **3.2 — Hoist magic numbers/strings to named constants**
  - File: cross-file (cli, installer, library, updater, security-auditor, formatter, config, uninstaller)
  - Principle: Magic numbers / strings
  - Effort: ~3h
  - Depends on: none
  - Acceptance: timeouts, short-hash, SHA regex, security thresholds, formatter widths, `EMPTY_CELL`, `.DS_Store` are named constants.

- [ ] **3.3 — Delete dead code (imports, fields, helpers, no-op blocks)**
  - File: `src/evaluator.ts:38,1310,2026`, `src/formatter.ts:25`, `src/installer.ts:976`
  - Principle: Dead code / clutter
  - Effort: ~1h
  - Depends on: none
  - Acceptance: `tsc`/lint show no new unused symbols; behavior unchanged.

- [ ] **3.4 — Fix negative conditionals & misleading names**
  - File: `src/installer.ts:81`, `src/library.ts:431`, `src/evaluator.ts:1455`
  - Principle: Positive conditionals; descriptive names
  - Effort: ~1h
  - Depends on: none
  - Acceptance: `hasSeparator` guard; `_overrides`→`overrides`; `detectGitAuthor` scope matches its name/comment.

- [ ] **3.5 — De-duplicate filename/marker conventions & encode provider+scope as a value object**
  - File: `src/ingester.ts:227`, `src/uninstaller.ts:99,198`
  - Principle: DRY; primitive obsession
  - Effort: ~2h
  - Depends on: none
  - Acceptance: `indexFileName()` + `MARKER_PREFIXES`/`buildMarker()`; provider+scope key no longer string-split.

- [ ] **3.6 — Encapsulate nested-ternary path resolution & remove shipped TODO**
  - File: `src/updater.ts:473,583`
  - Principle: Encapsulate conditionals; inappropriate information
  - Effort: ~1h
  - Depends on: none
  - Acceptance: `resolveConfiguredPath` helper with early returns; TODO implemented or moved to a tracked issue.

## Notes

- **Scope sampling.** This audit covers the 10 highest-value files (largest + highest-churn) — ~14.5K of ~24.7K LOC. The other 48 files (TUI views, small utils, eval providers, registry, doctor, publisher, scanner, stats, exporter, importer, bundler, skill-state) were **not** audited; `scanner.ts` (703) and `doctor.ts` (734) are the largest unaudited files and are the obvious next batch.
- **Cross-file patterns** were promoted to single findings (each enumerates its sites) to avoid double-counting; an individual site is reported **either** as a standalone finding **or** under its pattern, never both. The six recurring patterns: (A) swallowed-exception catch-alls, (B) God functions around clone/audit/swap, (C) error-message coalescing duplication, (D) scattered magic numbers, (E) `_overrides` test-seams in production signatures, (F) integration-tests-as-unit-tests.
- **The three security-auditor bugs (1.1–1.3) were verified by direct source read**, not just agent report — they are genuine false-negative paths in the tool whose purpose is to catch dangerous skills, and should be treated as the top priority.
- **TDD/ATDD half.** Every audited source file has a `.test.ts` sibling and CI runs unit/build/e2e, so there is a safety net — but the suite leans on real binaries (`git`/`npx`), global `process.chdir`, and subprocess invocation of the CLI (an **inverted pyramid** for `cli.ts`, whose handlers have no in-process unit tests). `cli.test.ts` was sampled for structure (subprocess wrapper, 354 tests, 37 conditional/loop constructs) but its individual assertions were not line-audited.
- **Pre-existing `tsc` failures** in `installer.ts` exist on `main` independent of this audit; CI (`ci.yml`) runs unit-tests/build/e2e but not `tsc`, so type regressions can slip through — consider adding `typecheck` to CI as a follow-up (not scheduled here as it's outside the clean-code rubric).
- **No source files were modified by this audit.**
- **Sustaining practices** (cheat sheet): commit/code reviews against these guidelines and pairing on the Phase 1 security fixes would prevent the `catch {}` and God-function patterns from re-accumulating.
