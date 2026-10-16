# Verification — gradle-upgrade / slice 3

> Builder model: Opus 5.5 (claude-opus-5-5, `builder-gradle-upgrade-3`) · Verifier model: Fable 5.1 (claude-fable-5-1, `verifier-gradle-upgrade-3`)
> Every number below is the observed output of a command named next to it. Never assumed,
> never copied from the builder.

Commit under test: `e12f777a515b8119fe2d795be9bbd51bc529b07c` (`git rev-parse HEAD`), the last
slice; parent `8e2b0d1` (verify-2). Base `46cd9856e9d589a2ce56ad2006bb20b7599372da`. All commands
run from the worktree `.claude/worktrees/gradle-upgrade` on branch `feature/gradle-upgrade`
(`git branch --show-current`). `J21=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home`
(`$J21/bin/java -version` → `openjdk version "21.0.12.1" 2026-08-18`),
`J27=/opt/homebrew/opt/openjdk@27/libexec/openjdk.jdk/Contents/Home` (`openjdk version "27" 2026-09-15`).
Pre-state: `git status --porcelain | wc -l` → `0`; `ls ~/.gradle/jdks` → `No such file or directory`.
The primary checkout was not entered or modified. The roster is `strict`, so there is no separate
feature reviewer: § Whole branch below covers the branch `46cd985..HEAD` as a whole.

## Mechanical gate

`AGENTS.md § Mechanical gate` (as rewritten by this slice) has one command, `./gradlew test`. Run
with `clean` first and `--warning-mode all`, as the contract's acceptance 1 and the design's slice-3
table prescribe. First Gradle run of this session, on the worktree as committed.

| Command | Result |
|---|---|
| `JAVA_HOME=$J21 ./gradlew clean test --warning-mode all > gate3v.log 2>&1; echo "exit=$?"` | **pass, `exit=0`**, `BUILD SUCCESSFUL in 637ms`, `4 actionable tasks: 4 executed` |
| `grep -inE 'deprecat\|warning' gate3v.log; echo "(grep exit=$?)"` | no line, `grep exit=1` |
| `grep -c 'Deprecated Gradle features' gate3v.log` | `0` |
| `grep -h '<testsuite' build/test-results/test/TEST-*.xml` | `CompareTasksTest tests="1"`, `TodoListTest tests="6"`, `TaskTest tests="7"`, each `skipped="0" failures="0" errors="0"`, timestamps `2026-10-16T14:21:03.165Z` / `.180Z` / `.175Z`; `ls -l` shows the three XML files written `Oct 16 17:21` (fresh, after `clean`) |
| summed per attribute by `grep -ho '<attr>="[0-9]*"' build/test-results/test/TEST-*.xml \| grep -o '[0-9]*' \| paste -sd+ - \| bc` | `tests=14`, `failures=0`, `errors=0`, `skipped=0` |
| `git status --porcelain \| wc -l` immediately after | `0` |
| `ls ~/.gradle/jdks` after | `No such file or directory` (no JDK provisioned) |

Verbatim gate output (`cat gate3v.log`):

```
> Task :clean

> Task :compileJava
Note: /Users/ituser/Documents/tdli-ws/todo-list-java/.claude/worktrees/gradle-upgrade/src/main/java/TodoList/FileReader.java uses unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.

> Task :processResources NO-SOURCE
> Task :classes
> Task :compileTestJava
> Task :processTestResources NO-SOURCE
> Task :testClasses
> Task :test

[Incubating] Problems report is available at: file:///Users/ituser/Documents/tdli-ws/todo-list-java/.claude/worktrees/gradle-upgrade/build/reports/problems/problems-report.html

BUILD SUCCESSFUL in 637ms
4 actionable tasks: 4 executed
Consider enabling configuration cache to speed up this build: https://docs.gradle.org/9.7.1/userguide/configuration_cache_enabling.html
```

The gate was run a **second time at the end** of this verification, after the JDK 27 runs
(§ Whole branch), so `build/` ends in the JDK 21 state: `exit=0`, `BUILD SUCCESSFUL in 539ms`, no
`deprecat`/`warning` line (`grep exit=1`), `tests=14 failures=0 errors=0 skipped=0`,
`git status --porcelain | wc -l` → `0`.

## Invariants

From `AGENTS.md § Invariants`. Slice 3 is docs-only, so the four code invariants reduce to one
empty diff over the whole branch; `BUILD-1` is the invariant this slice rewrites, so it is checked
against what was observed.

| Code | Checked how | Result |
|---|---|---|
| `SER-1` | `git diff 46cd985..HEAD -- src/ objectFile.txt dataFile.txt settings.gradle codegraph.toml scip-java.json .idea/ \| wc -c` | `0`: `Task.java` untouched on the branch | pass |
| `IO-1` | same diff (`FileReader.java`, `objectFile.txt`); `git status --porcelain` after every Gradle run of this session (gate ×2, `build`, `tasks`, `help`, `dependencies`, two JDK 27 runs, two `-P` probes) | diff `0`; status `0` every time; `objectFile.txt` never appears | pass |
| `UI-1` | same `src/` diff; no `application`/`run` (`grep -nE 'application\|mainClass\|compile\|enable-preview\|FizzBuzz\|useJUnitPlatform\|sourceCompatibility' build.gradle` → only `25:    useJUnitPlatform()`; `./gradlew -q help --task run` → `exit=1`, `Task 'run' not found in root project 'todolist'.`) | pass |
| `DATE-1` | `TodoListTest tests="6"` green in the fresh XML; `TodoList.java:16` in the empty `src/` diff | pass |
| `BUILD-1` (new text, `AGENTS.md:38`) | each clause re-observed: `grep -n distributionUrl gradle/wrapper/gradle-wrapper.properties` → `3:distributionUrl=https\://services.gradle.org/distributions/gradle-9.7.1-bin.zip`; `sed -n '8,12p' build.gradle` → the `java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }` block; `cat gradle.properties` → `org.gradle.java.installations.auto-download=false`; `JAVA_HOME=$J21 ./gradlew -q javaToolchains` → `Auto-download: Disabled`, one toolchain `Homebrew JDK 21 (21.0.12.1)` `Detected by: Current JVM`; gate green on JDK 21 (above); JDK 27 runs A and B below reproduce "green only with `-Dorg.gradle.java.installations.paths=<JDK 21>`, otherwise `Cannot find a Java installation … matching: {languageVersion=21 …}`" | pass: every clause matches an observation; no JDK other than 21 and 27 is named |

## Design and contract conformance

**Files and commit.** `git diff --name-only HEAD~1 HEAD` → `AGENTS.md`, `docs/todo-list/README.md`,
`docs/todo-list/evidence/2026-10-16-gradle-upgrade.md`, `readme.md` (4 files, the design's slice-3
list). `git log -1 --format='[%s]'` → `[gradle-upgrade slice 3: docs state what was observed]`, the
design's slice-3 title verbatim; trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
`git diff HEAD~1 HEAD --check` → exit `0` (no whitespace errors introduced); `grep -c $'\r' AGENTS.md readme.md`
→ `0`, `0`.

**The design's slice-3 verification table, re-run:**

| Command | Observed |
|---|---|
| `grep -nE 'Gradle 6\|6\.7\|JDK 15\|≤ 15\|15 or older\|FizzBuzz\|gradle run\|gradle test\|committed, see' AGENTS.md readme.md` | no match (`grep exit=1`) |
| `grep -rnE '6\.7\|JDK 15\|JDK ≤ 15\|FizzBuzz\|gradle run\|committed' AGENTS.md readme.md docs/` minus the contract, design, evidence file and verify reports | exactly four hits: `AGENTS.md:27` ("(committed)" on the `objectFile.txt` row; `git ls-files objectFile.txt` → tracked, so true), `docs/todo-list/README.md:8` (the contract's one-line description of what it set out to change), `docs/todo-list/evidence/2026-10-16-codegraph-onboarding.md:70` and `:72` (a dated record of the base). None is a current claim about the build. Same four the builder lists |
| `sed -n '8,12p;18,26p;30p' build.gradle` | `:8-12` the toolchain block; `:18-26` `dependencies { … }` (three lines, launcher included), blank, `test { useJUnitPlatform() }`; `:30` `        attributes "Main-Class": "Main.Main"`. Every cited range points at what the docs name |
| `sed -n '1p' .gitignore` | `build/` (cited as `.gitignore:1` at `AGENTS.md:60`) |
| `sed -n '3p' gradle/wrapper/gradle-wrapper.properties` | the 9.7.1 `distributionUrl` (cited at `AGENTS.md:38`) |
| `ls -d /opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home` | exists (the path quoted at `AGENTS.md:59`, `readme.md:26`) |
| `git diff 46cd985 HEAD -- AGENTS.md \| grep '^[-+]##'` | no output (`grep exit=1`): no heading changed |
| `git diff 46cd985 HEAD -- AGENTS.md \| grep '^[-+]@'` | no output: the import line unchanged |
| `diff <(git show 46cd985:AGENTS.md \| grep -E '^(## \|@)') <(grep -E '^(## \|@)' AGENTS.md) && echo HEADINGS-IDENTICAL` | `HEADINGS-IDENTICAL`; `grep -cxE '## (Subsystems\|Invariants\|Mechanical gate\|Branching)' AGENTS.md` → `4`; `head -1 AGENTS.md \| xxd` → `402e 2e2f …` = `@../dev-agent-kit/WORKSPACE.md` + LF |
| `bin/check` | not run (brief; `ls ../dev-agent-kit/WORKSPACE.md` from the worktree → `No such file or directory`, design R16). Its four heading checks emulated with the script's own `awk` (`dev-agent-kit/bin/check:33-36`, read-only): `## Subsystems (13 lines)`, `## Invariants (5 lines)`, `## Mechanical gate (5 lines)`, `## Branching (4 lines)`, all non-empty and the same counts as the base; import line present; `cmp docs/documentation-convention.md <kit>/docs/documentation-convention.md` → identical; `.mcp.json` registers `codegraph`. The Lead runs the real `bin/check` on the merged commit |
| `git diff 46cd985..HEAD -- src/ objectFile.txt dataFile.txt settings.gradle codegraph.toml scip-java.json .idea/ \| wc -c` | `0` |
| mechanical gate | green, 14/0/0/0 (§ Mechanical gate) |

**`AGENTS.md` stays a map.** `wc -l`: base `71`, HEAD `71`. Prose lines (not blank, table, bullet,
heading, fence, import or code line): base `17`, HEAD `17`. `git diff 46cd985 HEAD -- AGENTS.md`
→ `14` lines added, `13` removed, all in-place rewrites of the lines the design names (`:13-14`,
`:26`, `:38`, `:43`, `:46-47`, `:52-54`, `:59-60`, `:65-67`); no paragraph added.

**Text against the design's prescribed wording** (design § Slice 3 tables): `:13-14`, `:26`, `:43`,
`:46`, `:52-54`, `:59`, `:62`, `:65-67` and all three readme bullets match the design verbatim.
Two deliberate deviations, both toward precision and both backed by observation:

- `:38` BUILD-1 says "so Gradle must find an installed JDK 21" where the design drafted "so a JDK 21
  must be installed", and fills the design's `<one clause, the observed result>` with the two-sided
  JDK 27 result from verify-2 (green with `-D…installations.paths`, otherwise the quoted error).
  Verify-2 § Verdict asked for exactly this correction ("a JDK 21 must be installed" is not enough on
  this machine). Correct.
- `:60` says `(ignored, .gitignore:1)` where the design drafted `(ignored, see .gitignore)`. The
  citation is right (`sed -n '1p' .gitignore` → `build/`).

**`readme.md`.** `grep -nE 'gradle run|gradle test|dataFile|JDK 15' readme.md` → no match; it no
longer promises `gradle run`. Its run instruction (`readme.md:38-42`: run from the root folder,
`java -cp build/classes/java/main Main.Main`, reads and writes `objectFile.txt` in the current folder)
matches `AGENTS.md:62` and `:65-66` and the invariant `IO-1`. Its build steps (`export JAVA_HOME=<a JDK 21>`,
`./gradlew test`, `./gradlew build`) match `AGENTS.md:59-61`. `readme.md:23` "the build compiles
for Java 21 and never downloads a JDK" is backed by `javap` major `65` and `Auto-download: Disabled`
(below). The class-diagram link target exists (`ls docs/todo-list/guides/` → `class-diagram.png`).
The "Clone the repository" list is numbered 1, 2, 4, 5 (`readme.md:46-56`); `git show 46cd985:readme.md`
shows the same numbering at base, so it is pre-existing and outside this feature.

**Evidence file, slice-3 section.** Every line the builder's claim-to-evidence table cites was opened
with `sed -n '<n>p'` on the file as committed and says what the table says it says: evidence `:4-5`
(J21 path), `:21` (`openjdk version "21.0.12.1"`), `:98` (`.gitignore:1:build/`), `:127` (`build/$`),
`:361` (`junit-jupiter-api:5.6.0`), `:466` (`Gradle 9.7.1`), `:485` (`distributionUrl` line 3),
`:496` (`Auto-download: Disabled`), `:538-561` (`BUILD SUCCESSFUL` through the three `<testsuite` lines),
`:570-571` (`major version: 65` ×2), `:604` (`Main-Class: Main.Main`), `:666` (no `~/.gradle/jdks`);
verify-2 `:11`, `:27`, `:31`, `:67`, `:69`, `:75`, `:78`, `:86`, `:88`, `:111-113`, `:173-185` (run A,
`exit=0`, 14/0/0/0), `:187-195` (run B, `exit=1`, the verbatim `Cannot find a Java installation …`
message), `:224-229` (the arrow-`switch` correction). The builder's own slice-3 gate numbers
(`exit=0`, 14/0/0/0, `grep exit=1` for `deprecat|warning`, status showing only ` M AGENTS.md` /
` M readme.md`) reproduce here (§ Mechanical gate; my status was `0` because the slice is committed).
The builder's `HEADINGS-IDENTICAL` / `4` reproduce. `docs/todo-list/README.md:14` carries the updated
evidence line; `:8-9` list the contract and design, `:15-16` the two verify reports; this report is
added at `:17` by this commit.

**`## Not verified here` completeness** against design § Evidence file ("must at least cover") and
the brief: starting the application (yes, with the SER-1 `objectFile.txt` case and R17); `java -jar`
on the built jar (yes, manifest only); the JDK 27 result (yes: it states plainly that the builder's
slice-2 attempt executed no tests and that the `AGENTS.md` facts come from verify-2's `clean` runs);
Gradle on any JDK other than 21 and 27, JDK 17 named (yes); wrapper checksum not pinned (yes);
`.idea/` still JDK 15 with both `path:line`s (yes; `grep` of `.idea/misc.xml:14`, `.idea/compiler.xml:4`
confirms them); codegraph stale comments `codegraph.toml:14` and `:19-20` (yes; `sed -n '14p;19,20p' codegraph.toml`
→ ".gradle/, .idea/ and build/ are committed in this repository" and "the wrapper pins Gradle 6.7,
which only runs on JDK 15 or older", both now stale, file frozen by decision 6); codegraph
`build_tool` switch deferred (yes); the contract's arrow-`switch` inaccuracy (yes); `bin/check` not
run (yes); JDK 21 discovery on other machines (yes, an addition). Nothing the design listed is missing.

**Deviations:** none of substance; the two wording deviations above are improvements the earlier
verify report asked for.

## Whole branch

`strict` roster: no feature reviewer, so this section reads `46cd985..HEAD` as a whole.

**Commit sequence** (`git log --format='%h [%s]' 46cd985..HEAD`, oldest last):
`e12f777` slice 3 · `8e2b0d1` slice 2 verify report (green) · `60835d1` slice 2 · `ead0855` slice 1
verify report (green) · `6c22c77` slice 1 · `d892696` contract and design (the Lead's precondition
commit, design § Shape). Exactly the design's three slices in order, one verify-report commit after
each, plus the precondition. Every slice subject is the design's slice title verbatim.

**Scope.** `git diff 46cd985..HEAD --stat` → `70 files changed, 2459 insertions(+), 1342 deletions(-)`.
`git diff --name-status 46cd985..HEAD | grep -cE '^D\s+(build|\.gradle)/'` → `55` (the untracked
build state). The remaining 15 paths (`git diff --name-status 46cd985..HEAD -- . ':!build' ':!.gradle'`):
`.gitignore` M, `AGENTS.md` M, `build.gradle` M, `docs/todo-list/README.md` M, the contract, design,
evidence file and two verify reports A, `gradle.properties` A, `gradle/wrapper/gradle-wrapper.jar` M,
`gradle/wrapper/gradle-wrapper.properties` M, `gradlew` M, `gradlew.bat` M, `readme.md` M. Every one is
in design § Settled decisions "Scope of change"; nothing outside it.
**Frozen files and `src/`:** `git diff 46cd985..HEAD -- src/ objectFile.txt dataFile.txt settings.gradle codegraph.toml scip-java.json .idea/ | wc -c`
→ `0` (SER-1, IO-1, UI-1, DATE-1, decision 6). `cat settings.gradle` → `rootProject.name = 'todolist'`.

**Settled decisions 1-6, each with the observation on the branch head:**

| # | Decision | Observed |
|---|---|---|
| 1 | Gradle 9.7.1 | `JAVA_HOME=$J21 ./gradlew --version` → `Gradle 9.7.1`, `Launcher JVM: 21.0.12.1 (Homebrew 21.0.12.1)`; `gradle-wrapper.properties:3` → `gradle-9.7.1-bin.zip`; `ls ~/.gradle/wrapper/dists/gradle-9.7.1-bin/` → one directory `1w1c7tv4s851m17nbqdsro2tv` before and after all runs (cached distribution, no second download) |
| 2 | Java 21 toolchain, no JDK auto-provisioning | `build.gradle:8-12` toolchain block; `gradle.properties` = the design's two lines; `javaToolchains` → `Auto-download: Disabled`; `$J21/bin/javap -v … Main.Main \| grep 'major version'` → `65`, same for `TodoList.TaskTest`; `ls ~/.gradle/jdks` → `No such file or directory` before and after every run including the JDK 27 ones |
| 3 | `Main-Class` = `Main.Main`, no `compile` block, no `application`/`run` | `JAVA_HOME=$J21 ./gradlew build --warning-mode all` → `exit=0`, `> Task :jar`, `BUILD SUCCESSFUL in 256ms`, no `deprecat`/`warning` line; `unzip -p build/libs/todolist-1.0-SNAPSHOT.jar META-INF/MANIFEST.MF` → `Main-Class: Main.Main`; `unzip -l … \| grep -c '\.class$'` → `6`; `grep` of `build.gradle` → only `25:    useJUnitPlatform()`; `./gradlew -q tasks --all` lists `29` tasks, none named `run` (the only lines containing "run" are `tasks`, `check`, `test` descriptions); `help --task run` → `Task 'run' not found` |
| 4 | `build/`, `.gradle/` untracked; five-line `.gitignore`; `.idea/` untouched | `git ls-files build .gradle \| wc -l` → `0`; `git ls-files -ci --exclude-standard \| wc -l` → `0`; `cat -vet .gitignore` → `build/$ .gradle/$ .DS_Store$ bin/$ out/$`; `.idea/` in the empty frozen diff (`.idea/misc.xml:14` still `JDK_15`, `.idea/compiler.xml:4` still `target="15"`, as decision 4 says) |
| 5 | JUnit api 5.6.0 kept, versionless engine, launcher added versionless | `build.gradle:19-21`; `JAVA_HOME=$J21 ./gradlew -q --offline dependencies --configuration testRuntimeClasspath` → `exit=0`, `junit-jupiter-api:5.6.0`, `junit-jupiter-engine -> 5.6.0`, `junit-platform-launcher -> 1.6.0`, under `org.junit:junit-bom:5.6.0`; `grep -c FAILED` → `0` (offline: nothing downloaded by this verification) |
| 6 | codegraph switch deferred, `codegraph.toml` and `scip-java.json` byte-identical | in the empty frozen diff; `cat scip-java.json` → `{ "sourceFiles": ["src/main/java", "src/test/java"] }`; `codegraph.toml:23` still `java = { build_tool = "scip" }` |

**Contract acceptance 1-5, observed on the branch:**

1. Gate on JDK 21 with `--warning-mode all`: `exit=0`, `tests=14 failures=0 errors=0 skipped=0` from
   fresh XML, no deprecation or warning line (§ Mechanical gate, twice).
2. `./gradlew build` on JDK 21: `exit=0` (decision 3 row).
3. `git diff 46cd985..HEAD -- src/ objectFile.txt dataFile.txt` (within the frozen diff) → `0` bytes.
4. JDK 27 attempted and recorded. Re-observed here with `clean` each time:
   **(B)** `JAVA_HOME=$J27 ./gradlew clean test --warning-mode all` → `exit=1`, `BUILD FAILED in 282ms`,
   `> Cannot find a Java installation on your machine (Mac OS X 27.0.1 aarch64) matching: {languageVersion=21, vendor=any vendor, implementation=vendor-specific, nativeImageCapable=false}. Toolchain auto-provisioning is not enabled.`
   **(A)** `JAVA_HOME=$J27 ./gradlew clean test --warning-mode all -Dorg.gradle.java.installations.paths=$J21`
   → `exit=0`, `> Task :test` executed, `BUILD SUCCESSFUL in 1s`, `4 actionable tasks: 4 executed`, no
   `deprecat`/`warning` line, fresh XML `tests=14 failures=0 errors=0 skipped=0` (timestamp
   `2026-10-16T14:21:58.050Z`), `javap` of `TodoList.TaskTest` after this run → `major version: 65`
   (the Java 21 toolchain did the work). `ls ~/.gradle/jdks` after both → `No such file or directory`.
   This is exactly what `AGENTS.md:38` states.
5. `AGENTS.md` BUILD-1, Mechanical gate, Build/run/test and `readme.md` match what was observed
   (§ Invariants, § Design and contract conformance).

## Adversarial probes

1. **A claim that overstates the observation.** `grep -nE 'JDK 17|Java 17|17\+|Gradle 8|JDK 2[0-9]|Java 2[0-9]|auto-provision|download|java -jar|gradle run|dataFile|\bCI\b' AGENTS.md readme.md`
   → hits only the intended statements: `9.7.1`, `Java 21`/`JDK 21`, `no JDK auto-download`,
   "never downloads a JDK", `no CI`, the `dataFile.txt` subsystem row (unchanged legacy note) and the
   JDK 27 clause of BUILD-1. No JDK range ("17+", "21 or newer"), no "works on 27" without the
   installations path, no claim that `java -jar` starts the app (`AGENTS.md:67` says the opposite:
   "not verified by agents"). Not broken. One precision note, not a defect: BUILD-1 says JDK 27 is
   green "only with `-D…installations.paths`"; verify-2 run C showed the deprecated `-P` form also
   works in 9.7.1 with a Gradle 10 deprecation warning. The `-D` form is the one Gradle recommends and
   the one the docs should name; "only" is true for the non-deprecated routes observed.
2. **The Mechanical gate line omits `--warning-mode all`: would a future deprecation go unnoticed?**
   The design prescribed the line without the flag (design § Slice 3, `:43` row, verbatim), and the
   contract keeps `./gradlew test` as the gate with `--warning-mode all` as the verifier's acceptance
   check (contract § Acceptance 1), so the omission is as designed. Observed effect: with a known
   deprecation trigger on the command line (`JAVA_HOME=$J21 ./gradlew test -Porg.gradle.java.installations.paths=$J21`,
   no flag) → `exit=0` and the summary `Deprecated Gradle features were used in this build, making it
   incompatible with Gradle 10.` plus `You can use '--warning-mode all' …` is printed (lines 9 and 11
   of the log); the same command with `--warning-mode all` prints the individual
   `Specifying 'org.gradle.java.installations.paths' … has been deprecated` line instead. So a future
   deprecation is still announced by the documented command; only its location is hidden until the
   flag is added. Non-blocking; a Ship-gate note.
3. **A `path:line` that points at the wrong thing.** Every citation in the new text opened
   (§ Design conformance table): `build.gradle:8-12`, `:18-26`, `:30`, `gradle-wrapper.properties:3`,
   `.gitignore:1`, the Homebrew path. All correct.
4. **Headings or import touched** (`bin/check` would fail at merge). Diff of `^[-+]##` and `^[-+]@`
   lines empty; `HEADINGS-IDENTICAL`; emulated body-line counts equal the base's. Not broken.
5. **`readme.md` still telling users `gradle run` or something now wrong.** `gradle run`/`gradle test`
   gone; no `dataFile.txt` mention; the `java -cp` path exists after `./gradlew build`
   (`ls build/classes/java/main/Main/Main.class` → present). The only oddity is the pre-existing
   1, 2, 4, 5 numbering in "Clone the repository" (unchanged since base; out of scope).
6. **`.idea/` or `codegraph.toml` edited "for consistency".** Both in the empty frozen diff. The stale
   `codegraph.toml` comments are listed under "Not verified here" with the right line numbers.
7. **Evidence numbers copied rather than observed.** The builder's slice-3 gate and grep outputs all
   reproduce (above); every cited evidence/verify-2 line says what the claim table says.
8. **Slice reverts cleanly on its own.** `git revert --no-commit HEAD` → `exit=0`; staged paths exactly
   the four slice files; `git diff --cached --quiet HEAD~1 -- AGENTS.md readme.md` → identical to the
   slice-2 state; `git revert --abort` → `exit=0`; `git status --porcelain | wc -l` → `0`; HEAD still
   `e12f777`.
9. **Worktree dirtied by any run.** `git status --porcelain | wc -l` → `0` after the gate, after
   `build`, after both JDK 27 runs, after the `-P` probes and after the final gate.

## Verdict

**green** — the mechanical gate on the branch head reproduces on JDK 21 (`exit=0`, 14 tests,
0 failures, 0 errors, 0 skipped, no deprecation or warning line, worktree clean); every factual
claim in the new `AGENTS.md` and `readme.md` text matches a line of the evidence or verify reports
and was re-observed here, including both JDK 27 outcomes; every `path:line` citation points at what
it names; the four headings and the import line are byte-identical to base and `AGENTS.md` stays a
71-line map; `readme.md` no longer promises `gradle run` and its run instruction matches `AGENTS.md`;
the README lists the contract, design, evidence file and verify reports; the commit subject is exact
and touches only the four named files. Whole branch: frozen files and `src/` unchanged (`0` bytes of
diff), settled decisions 1-6 each honoured with an observation, acceptance 1-5 each observed, commit
sequence as designed. No unsettled decision was touched.

For the Ship gate, two non-blocking notes: the documented gate command has no `--warning-mode all`
(as designed; a future deprecation still shows as Gradle's summary line, probe 2), and the stale
`codegraph.toml:14` / `:19-20` comments stay for the deferred codegraph follow-up (decision 6).
