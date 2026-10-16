# Verification — gradle-upgrade / slice 1

> Builder model: Opus 5.5 (claude-opus-5-5, `builder-gradle-upgrade-1`) · Verifier model: Fable 5.1 (claude-fable-5-1, `verifier-gradle-upgrade-1`)
> Every number below is the observed output of a command named next to it. Never assumed,
> never copied from the builder.

Commit under test: `6c22c77bdf6581878597cd46e5800a19dea38193` (`git rev-parse HEAD`), parent
`d892696dd590452e6a6e040e0b2607f10a88f7ea` (`git log -1 --format=%P HEAD`), base
`46cd9856e9d589a2ce56ad2006bb20b7599372da`. All commands run from the worktree
`.claude/worktrees/gradle-upgrade` on branch `feature/gradle-upgrade` (`git branch --show-current`).
`J21=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home`;
`$J21/bin/java -version 2>&1 | head -1` → `openjdk version "21.0.12.1" 2026-08-18`.
The primary checkout was not entered or modified.

## Mechanical gate

`AGENTS.md § Mechanical gate` has one command, `./gradlew test`. The design (§ Slice 1,
verification table) expects it **red on this slice** with the base's failure, because the wrapper
is still Gradle 6.7 until slice 2; the pass condition for slice 1 is that the observed failure is
exactly that one, not a green run.

| Command | Result |
|---|---|
| `JAVA_HOME=$J21 ./gradlew test > gate.log 2>&1; echo "exit=$?"` | **fail, `exit=1`** (expected on this slice) |
| `wc -l < gate.log` | `233` |
| `grep -n -m3 'Unsupported class file major version' gate.log` | `10:  General error during semantic analysis: Unsupported class file major version 65` / `12:  java.lang.IllegalArgumentException: Unsupported class file major version 65` |
| `git status --porcelain \| wc -l` immediately after the gate | `0` (the gate left the worktree clean) |

First failure lines, verbatim (`head -12 gate.log`, first line of the file is blank):

```
FAILURE: Build failed with an exception.

* Where:
Settings file '/Users/ituser/Documents/tdli-ws/todo-list-java/.claude/worktrees/gradle-upgrade/settings.gradle'

* What went wrong:
Could not compile settings file '/Users/ituser/Documents/tdli-ws/todo-list-java/.claude/worktrees/gradle-upgrade/settings.gradle'.
> startup failed:
  General error during semantic analysis: Unsupported class file major version 65
  
  java.lang.IllegalArgumentException: Unsupported class file major version 65
```

Last line (`tail -1 gate.log` after the `* Get more help` block): `BUILD FAILED in 276ms`.

**Is this the failure the design predicts?** Yes. Design § Slice 1 step 3 and the verification
table say "red, `Unsupported class file major version 65` (same as base; slice 1 does not touch
the build)", and contract:184-187 / `docs/todo-list/evidence/2026-10-16-codegraph-onboarding.md:70-72`
record the same message on the base. The failure is at settings-script compilation, before any
task runs, so no test count exists to report (0 tests executed). Gate on slice 1: **pass** (red as
predicted, nothing else).

## Invariants

Codes from `AGENTS.md § Invariants`; the contract (§ Invariants to preserve) lists the same five.
Slice 1 changes no Java file, so SER-1, IO-1, UI-1 and DATE-1 are checked by one empty diff.

| Code | Checked how | Result |
|---|---|---|
| `SER-1` | `git diff 46cd985..HEAD -- src/ \| wc -l` (part of the frozen-file diff below) | `0`: `Task.java` untouched · pass |
| `IO-1` | same diff for `src/main/java/TodoList/FileReader.java`; `objectFile.txt` in the frozen diff; `git status --porcelain` after the gate | diff `0` lines, status `0` lines: persistence code and the data file unchanged, the gate wrote nothing tracked · pass |
| `UI-1` | same `src/` diff (`UserInterface.java`, `TodoList.java`) | `0` lines · pass |
| `DATE-1` | same `src/` diff (`TodoList.java:16`, `src/test/.../TodoListTest.java`) | `0` lines; no test edited · pass |
| `BUILD-1` | the gate above; `git diff 46cd985..HEAD -- gradle/ gradlew gradlew.bat build.gradle settings.gradle \| wc -l` | still Gradle 6.7, still fails on JDK 21 with "Unsupported class file major version" as the invariant states; wrapper diff `0` lines · pass (unchanged on this slice; rewritten by slice 3 after slice 2 makes it false) |

Frozen-file diff, the one command behind the rows above:
`git diff 46cd985..HEAD -- src/ objectFile.txt dataFile.txt settings.gradle build.gradle gradle/ gradlew gradlew.bat codegraph.toml scip-java.json .idea/ gradle.properties readme.md AGENTS.md | wc -l` → `0`.
`git ls-files -s gradlew` → `100755 4f906e0c811fc9e230eb44819f509cd0627f2600 0	gradlew` (mode kept).

## Design and contract conformance

Against design § Settled decisions 4 and 6, § Interfaces → `.gitignore`, § Slice 1, § Evidence
file, and contract § Invariants / § Acceptance.

**`.gitignore` byte for byte.** `printf 'build/\n.gradle/\n.DS_Store\nbin/\nout/\n' > gitignore.design`
(the design's exact content), then:

| Command | Observed |
|---|---|
| `cmp gitignore.design .gitignore && echo IDENTICAL` | `IDENTICAL` |
| `git show HEAD:.gitignore \| cmp - gitignore.design && echo IDENTICAL-HEAD` | `IDENTICAL-HEAD` (index = working tree = design) |
| `shasum -a 256 .gitignore gitignore.design` | both `5834edbb5e89306531c6c4421b133c59693e450891489b6330eab0c63553888c` |
| `wc -c .gitignore` / `wc -l .gitignore` | `36` bytes / `5` lines |
| `xxd .gitignore` | `6275 696c 642f 0a2e …` : LF only, no BOM, no leading `/`, ends with `0a` |
| `grep -c $'\r' .gitignore` / `grep -nE '[[:space:]]$' .gitignore` | `0` / no output |

**Exactly the base's `build/` and `.gradle/` files removed, nothing else.**

| Command | Observed |
|---|---|
| `git diff --name-status HEAD~1 HEAD \| wc -l` | `58` |
| `… \| grep -cE '^D\s+(build\|\.gradle)/'` | `55` |
| `… \| grep -cE '^D\s+build/'` / `… '^D\s+\.gradle/'` | `25` / `30` |
| `… \| grep -vE '^D'` | `M .gitignore`, `M docs/todo-list/README.md`, `A docs/todo-list/evidence/2026-10-16-gradle-upgrade.md` — nothing else |
| `… \| grep -E '^D' \| grep -vE '^D\s+(build\|\.gradle)/'` | no output (no deletion outside the two directories) |
| `git ls-tree -r --name-only 46cd985 -- build .gradle \| sort > base_bg.txt; wc -l < base_bg.txt` | `55` |
| `diff base_bg.txt <(… '^D' … \| cut -f2 \| sort) && echo SETS-IDENTICAL` | `SETS-IDENTICAL` (same against the parent `d892696`: `PARENT-SETS-IDENTICAL`) |
| `git ls-files build .gradle \| wc -l` | `0` |
| `git ls-tree -r --name-only 46cd985 \| wc -l` / `… d892696 …` / `git ls-files \| wc -l` | `93` / `95` / `41` = 95 − 55 + 1 |

**No tracked path is ignored; the new patterns catch nothing tracked.**

| Command | Observed |
|---|---|
| `git ls-files -ci --exclude-standard \| wc -l` | `0` |
| `git ls-files \| grep -E '(^\|/)(bin\|out)/'` | no output |
| `git ls-files \| grep -iE '(^\|/)(build\|\.gradle\|bin\|out)/\|\.DS_Store'` | no output |
| `git check-ignore -v build/x .gradle/x bin/x out/x .DS_Store src/main/.DS_Store` | `.gitignore:1:build/ build/x`, `:2:.gradle/ .gradle/x`, `:4:bin/ bin/x`, `:5:out/ out/x`, `:3:.DS_Store .DS_Store`, `:3:.DS_Store src/main/.DS_Store` |
| `git check-ignore -v gradle/wrapper/gradle-wrapper.jar gradle/wrapper/gradle-wrapper.properties gradle/x gradlew gradlew.bat gradle.properties .idea/misc.xml .idea/compiler.xml src/main/java/Main/Main.java src/test/java/TodoList/TaskTest.java objectFile.txt dataFile.txt build.gradle settings.gradle codegraph.toml scip-java.json readme.md AGENTS.md` | no output for any of them (`gradle/` is not caught by `.gradle/`; `build.gradle` is not caught by `build/`) |

**Files remain on disk.** `test -d build && test -d .gradle && echo present` → `present`;
`find build -type f | wc -l` → `25`; `find .gradle -type f | wc -l` → `30`. Per-file check of all
55 base-tracked paths (`git ls-tree -r 46cd985 -- build .gradle | awk '{print $3, $4}'`, then
`test -f` and `git hash-object` against the base blob): `53` `SAME`, `0` `MISSING`, `2` `CHANGED`:
`.gradle/6.7/fileHashes/fileHashes.lock` and `.gradle/checksums/checksums.lock`, both rewritten by
the Gradle 6.7 gate runs (`find .gradle build -type f -newer .gitignore` lists them plus
`.gradle/6.7/fileChanges/last-build.bin`). Nothing was deleted from disk; the two changed lock
files are the live demonstration of why the directory is untracked, and `git status --porcelain`
stays empty (`git status --porcelain --ignored` → exactly `!! .gradle/` and `!! build/`).

**Evidence file** (`docs/todo-list/evidence/2026-10-16-gradle-upgrade.md`), each number re-observed:

| Claim in the evidence file | My observation |
|---|---|
| `openjdk version "21.0.12.1" 2026-08-18` | same (`$J21/bin/java -version`) |
| gate `exit=1`, `Unsupported class file major version 65`, `BUILD FAILED` | same; only the timing (`1s` vs `276ms`) and the "Starting a Gradle Daemon" line differ, both environmental |
| `[… 211 lines elided …]` | `grep -n` of the `IllegalArgumentException` line (`12`) and the `  1 error` line (`225`) in my log: 225 − 12 − 2 = `211`. Kept head/tail lines identical to mine (`diff` of the kept blocks: one extra blank line after `1 error` in my log, nothing else). The elided region contains no line matching the design's keep-rule `grep -E 'deprecat\|warning\|FAILED\|BUILD\|[0-9]+\.[0-9]+\.[0-9]+'` → no output |
| `55` / `25` / `30` removals | same (table above) |
| `.gitignore` five LF lines | same (`xxd`, `wc -l`) |
| `git ls-files -ci --exclude-standard` empty | same |
| frozen diff empty | same |
| `git ls-files -s gradlew` → `100755 4f906e0c…` | same |
| `git status --porcelain` clean after the gate | same, re-observed after my own gate run |

Format: preamble names the worktree, branch, base and `JAVA_HOME` with the `java -version` line;
one `## Slice 1 — …` section; every block is headed `### \`<exact command>\`` followed by a fenced
verbatim block and one or two sentences (`grep -nE '^#|^```'` over the file: 15 command headings,
15 fenced pairs). It follows design § Evidence file and
`docs/documentation-convention.md` rule 4. The builder ran the design's `HEAD~1 HEAD` checks as
`git diff --cached … HEAD` before committing and says so in the file; I re-ran the committed form
above with identical results. Not a deviation of substance.

**README and commit message.** `grep -n 'gradle-upgrade' docs/todo-list/README.md` → line 14
`` - `evidence/2026-10-16-gradle-upgrade.md` — observed outputs of the gradle-upgrade slices (wrapper 9.7.1, gate on JDK 21, JDK 27 attempt). `` (the design's prescribed wording).
`git log -1 --format='[%s]' HEAD` → `[gradle-upgrade slice 1: untrack build state and fix .gitignore]`, exactly the required first line.
`git diff --name-status 46cd985..HEAD -- . ':!build' ':!.gradle'` → `.gitignore` (M), `docs/todo-list/README.md` (M), the contract, the design and the evidence file (A): nothing on the branch outside the design's scope.

**Settled decisions.** Decision 4 implemented as written (`.idea/` untouched, in the frozen diff).
Decision 6 respected: `codegraph.toml` and `scip-java.json` byte-identical. No other decision is in
slice 1's scope. Design § Decisions left to implementation is `None`; the slice met none.

**Deviations:** none.

## Adversarial probes

1. **Pattern too wide / anchoring.** `git check-ignore -v src/build/x docs/out/x src/test/bin/x build .gradle`
   → all five ignored (`.gitignore:1`, `:5`, `:4`, `:1`, `:2`). The patterns are unanchored, so a
   `build/`, `bin/` or `out/` directory at any depth is ignored. That is what the design specifies
   (its exact content has no leading `/`, and § Slice 1 lists "a leading `/` that changes
   matching" as a defect), and it catches nothing tracked (`git ls-files -ci --exclude-standard`
   → `0`; `git ls-files | grep -iE '(^|/)(build|\.gradle|bin|out)/'` → no output). `.gradle/`
   does not hit `gradle/` (negative list above). Not broken.
2. **Case-insensitive filesystem.** `git config core.ignorecase` → `true`; `git check-ignore -v Build/x BIN/x`
   → both ignored via `.gitignore:1` / `:4`. No tracked path has such a name (same `grep -i` as
   above), so no effect today; noted for anyone adding a `Build/` directory on this machine.
3. **A tracked file that depends on `build/` or `.gradle/` content.** `git grep -n -E 'build/|\.gradle' -- . ':!docs'`
   → hits only in `.gitignore` (the rules), `.idea/.gitignore` and `.idea/workspace.xml` (IDE state
   scoped to `.idea/`), `AGENTS.md:60` and `:62` (docs, rewritten by slice 3 per design § Slice 3),
   `build.gradle:28` / `gradlew` / `gradlew.bat` / `gradle-wrapper.properties` (the words
   `build.gradle` and `org.gradle`, not paths), and `codegraph.toml:13-15`. `cat scip-java.json` →
   `{ "sourceFiles": ["src/main/java", "src/test/java"] }`: the index reads sources only, and
   `codegraph.toml:15` excludes `.gradle/**`. Nothing tracked reads committed build output. One
   stale comment remains: `codegraph.toml:14` says ".gradle/, .idea/ and build/ are committed in
   this repository", now false for two of the three; the file is frozen and byte-identical by
   settled decision 6, so this is a note for the codegraph follow-up, not a slice-1 defect.
4. **Gate leaves the worktree dirty.** `git status --porcelain | wc -l` after my gate run → `0`,
   although the run rewrote two lock files and `last-build.bin` under `.gradle/6.7/` (probe in
   § Design conformance). Before slice 1 those would have shown as modified tracked files.
5. **Files deleted from disk instead of the index.** 55/55 present, 53 byte-identical to base
   (§ Design conformance). Not broken.
6. **CRLF, BOM, trailing whitespace, stray text in `.gitignore`.** `xxd`, `grep -c $'\r'` → `0`,
   `grep -nE '[[:space:]]$'` → none, `file .gitignore` → `ASCII text`. Not broken.
7. **`.idea/` touched.** `git diff 46cd985..HEAD -- .idea/ | wc -l` (in the frozen diff) → `0`.
8. **Revert restores the index cleanly.** Two ways, neither run in the feature worktree's index
   (the design's `git revert --no-commit HEAD … --abort` would re-add and then delete the ignored
   on-disk files here). (a) Temp index: `GIT_INDEX_FILE=tmp git read-tree HEAD; git diff --binary HEAD~1 HEAD | GIT_INDEX_FILE=tmp git apply --cached -R`
   → applied; `GIT_INDEX_FILE=tmp git ls-files build .gradle | wc -l` → `55`;
   `GIT_INDEX_FILE=tmp git write-tree` → `f7e7d7e5a3e80a85e1394e20e0ca45f53477c972` =
   `git rev-parse 'HEAD~1^{tree}'`. (b) Throwaway detached worktree in the scratchpad
   (`git worktree add --detach <scratch>/revert-probe HEAD`): `git revert --no-commit HEAD` →
   exit `0`; `git ls-files build .gradle | wc -l` → `55`; `git status --porcelain | wc -l` → `58`
   (the 55 re-added paths, `M .gitignore`, `M docs/todo-list/README.md`, `D` the evidence file);
   `git revert --abort` → exit `0`, status `0` lines, HEAD still `6c22c77`; worktree removed and
   pruned (`git worktree list` shows only the primary checkout and the feature worktree). The
   feature worktree afterwards: `git status --porcelain | wc -l` → `0`, HEAD `6c22c77…`.
9. **Evidence numbers copied rather than observed.** Every number re-observed (§ Design
   conformance); the elision count `211` reproduced from my own log.
10. **Leftover experiment directories committed** (`bin/`, `build/reports/problems/`,
    `.gradle/9.7.1/`, contract:215-218). `ls -d bin out` → neither exists in the worktree;
    `git ls-files | grep -E '(^|/)(bin|out)/'` → none; `find .gradle -type f | wc -l` → `30`, the
    base's set only. Not present.

## Verdict

green — the mechanical gate is red exactly as the design predicts for slice 1 (`exit=1`,
`Unsupported class file major version 65` from the Gradle 6.7 wrapper on JDK 21, same as base) and
leaves the worktree clean; `.gitignore` is byte-identical to the design; the 55 index removals are
exactly the base's `build/` and `.gradle/` files with nothing else removed, all still on disk;
no tracked path is ignored and the new patterns catch nothing tracked; every frozen file is
byte-identical to base; the evidence file's numbers all reproduce and it follows the design's
format; the README lists it; the commit subject is exact; the commit reverts cleanly. No unsettled
decision was touched. One non-blocking note for the codegraph follow-up: `codegraph.toml:14` now
describes `build/` and `.gradle/` as committed (frozen file, left alone by settled decision 6).
