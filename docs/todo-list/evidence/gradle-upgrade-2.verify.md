# Verification — gradle-upgrade / slice 2

> Builder model: Opus 5.5 (claude-opus-5-5, `builder-gradle-upgrade-2`) · Verifier model: Fable 5.1 (claude-fable-5-1, `verifier-gradle-upgrade-2`)
> Every number below is the observed output of a command named next to it. Never assumed,
> never copied from the builder.

Commit under test: `60835d1d7a05ee7e05a73b544548bbf699105b60` (`git rev-parse HEAD`), parent
`ead0855660f725f1552aeb2abe23fae14c37a9f7` (`git log -1 --format=%P HEAD`), base
`46cd9856e9d589a2ce56ad2006bb20b7599372da`. All commands run from the worktree
`.claude/worktrees/gradle-upgrade` on branch `feature/gradle-upgrade` (`git branch --show-current`).
`J21=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home`,
`J27=/opt/homebrew/opt/openjdk@27/libexec/openjdk.jdk/Contents/Home`;
`$J27/bin/java -version 2>&1 | head -1` → `openjdk version "27" 2026-09-15`. Pre-state before any
command of mine: `ls ~/.gradle/jdks` → `No such file or directory`; `ls ~/.gradle/gradle.properties`
→ `No such file or directory`; `ls ~/.gradle/init.d` → `No such file or directory`;
`env | grep -iE 'GRADLE|JAVA_TOOL_OPTIONS|_JAVA_OPTIONS|JDK_JAVA_OPTIONS'` → only `PWD`. The
primary checkout was not entered or modified; the stash was read with `git show`, never applied.

## Mechanical gate

`AGENTS.md § Mechanical gate` has one command, `./gradlew test`. Run as the design's slice-2 table
prescribes (`clean` first, `--warning-mode all`, `JAVA_HOME=$J21`). This is the **first** run of the
session, on the worktree as committed.

| Command | Result |
|---|---|
| `JAVA_HOME=$J21 ./gradlew clean test --warning-mode all > gate1.log 2>&1; echo "exit=$?"` | **pass, `exit=0`**, `BUILD SUCCESSFUL in 673ms`, `4 actionable tasks: 4 executed` |
| `grep -inE 'deprecat\|warning' gate1.log; echo "(grep exit=$?)"` | no line, `grep exit=1` |
| `grep -c 'Deprecated Gradle features' gate1.log` | `0` |
| `grep -h '<testsuite' build/test-results/test/TEST-*.xml` | `CompareTasksTest tests="1"`, `TodoListTest tests="6"`, `TaskTest tests="7"`, each `skipped="0" failures="0" errors="0"`, timestamps `2026-10-16T14:07:10.124Z`, `…139Z`, `…133Z` (fresh; `ls -l` shows the three XML files written `Oct 16 17:07`) |
| summed by `grep -ho '<attr>="[0-9]*"' build/test-results/test/TEST-*.xml \| … \| bc` for each attribute | `tests=14`, `failures=0`, `errors=0`, `skipped=0` |
| `git status --porcelain` immediately after | empty |

Verbatim gate output (`cat gate1.log`):

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

BUILD SUCCESSFUL in 673ms
4 actionable tasks: 4 executed
Consider enabling configuration cache to speed up this build: https://docs.gradle.org/9.7.1/userguide/configuration_cache_enabling.html
```

The gate was run a **second time at the end** of this verification, after the JDK 27 runs, so the
worktree's `build/` ends in the JDK 21 state (§ Adversarial probes, "JDK 27"): `exit=0`, same
output modulo `BUILD SUCCESSFUL in 612ms`, no `deprecat`/`warning` line (`grep exit=1`),
`tests=14 failures=0 errors=0 skipped=0`, timestamps `2026-10-16T14:09:43.9xxZ`,
`javap` major `65` for `Main.Main` and `TodoList.TaskTest`, `git status --porcelain` empty.

### Other gate-adjacent commands from the design's slice-2 table

| Command | Observed |
|---|---|
| `JAVA_HOME=$J21 ./gradlew --version` | `Gradle 9.7.1`, `Launcher JVM: 21.0.12.1 (Homebrew 21.0.12.1)`, `Daemon JVM: /opt/homebrew/Cellar/openjdk@21/21.0.12.1/libexec/openjdk.jdk/Contents/Home (no Daemon JVM specified, using current Java home)`, `exit=0` |
| `ls ~/.gradle/wrapper/dists/gradle-9.7.1-bin/` | `1w1c7tv4s851m17nbqdsro2tv` (one directory, before and after all runs) |
| `JAVA_HOME=$J21 ./gradlew -q javaToolchains` | `Auto-detection: Enabled`, `Auto-download: Disabled`; one toolchain, `Homebrew JDK 21 (21.0.12.1)`, `Language Version: 21`, `Detected by: Current JVM`; `exit=0` |
| `JAVA_HOME=$J21 ./gradlew -q dependencies --configuration testRuntimeClasspath` | `exit=0`; `junit-jupiter-engine -> 5.6.0`, `junit-platform-launcher -> 1.6.0`, `junit-platform-engine:1.6.0`, `junit-platform-commons:1.6.0`, all under `org.junit:junit-bom:5.6.0`; `grep -c FAILED` → `0` |
| `$J21/bin/javap -v -cp build/classes/java/main Main.Main \| grep 'major version'` | `major version: 65` |
| `$J21/bin/javap -v -cp build/classes/java/test TodoList.TaskTest \| grep 'major version'` | `major version: 65` |
| `xxd -s 6 -l 2 -p` on all 9 class files under `build/classes/java/{main,test}` | `0041` (= 65) for every one |
| `JAVA_HOME=$J21 ./gradlew build --warning-mode all; echo "exit=$?"` | `exit=0`, `> Task :jar` executed, `BUILD SUCCESSFUL in 243ms`, `4 actionable tasks: 1 executed, 3 up-to-date`; `grep -inE 'deprecat\|warning'` → no line |
| `unzip -p build/libs/todolist-1.0-SNAPSHOT.jar META-INF/MANIFEST.MF` | `Manifest-Version: 1.0` / `Main-Class: Main.Main` |
| `unzip -l build/libs/todolist-1.0-SNAPSHOT.jar \| grep -c '\.class$'` | `6`: `Main/Main.class`, `Main/UserInterface.class`, `TodoList/CompareTasks.class`, `TodoList/FileReader.class`, `TodoList/Task.class`, `TodoList/TodoList.class`; `10 files` total (two directories, `META-INF/`, manifest); jar `10421` bytes |
| `git status --porcelain` after `build` | empty |
| `ls ~/.gradle/jdks` after every run | `No such file or directory` (never created) |

## Invariants

From `AGENTS.md § Invariants`.

| Code | Checked how | Result |
|---|---|---|
| `SER-1` | `git diff $BASE..HEAD -- src/ objectFile.txt dataFile.txt settings.gradle codegraph.toml scip-java.json .idea/ \| wc -l` | `0`: `Task.java` and every other source untouched; nothing in `build.gradle` adds a compiler flag (`grep -nE 'compile\|enable-preview\|FizzBuzz\|useJUnitPlatform\|sourceCompatibility\|application\|mainClass' build.gradle` → only `25:    useJUnitPlatform()`) | pass |
| `IO-1` | `git status --porcelain` empty after every Gradle run (gate ×2, `build`, `wrapper --rerun-tasks`, three JDK 27 runs, two `--info` runs); `objectFile.txt` never appears | pass |
| `UI-1` | same `src/` diff `0`; no `application` plugin or `run` task (`grep` above) | pass |
| `DATE-1` | `TodoListTest tests="6"` green in the fresh XML; `TodoList.java:16` unchanged (`src/` diff `0`) | pass |
| `BUILD-1` | the slice makes the current text ("pins Gradle 6.7, runs only on JDK 15 or older") false by design; slice 3 rewrites it. Observed facts it must state: wrapper `9.7.1` (`grep -n distributionUrl gradle/wrapper/gradle-wrapper.properties` → `3:…gradle-9.7.1-bin.zip`), toolchain 21 (`build.gradle:8-12`, `sed -n '8,12p'`), auto-download disabled (`javaToolchains`), gate green on JDK 21, and the JDK 27 result exactly as in § Adversarial probes | changed as designed; docs pending (slice 3) |

## Design and contract conformance

**Files in the commit** (`git diff --name-only HEAD~1 HEAD`): `build.gradle`,
`docs/todo-list/evidence/2026-10-16-gradle-upgrade.md`, `gradle.properties`,
`gradle/wrapper/gradle-wrapper.jar`, `gradle/wrapper/gradle-wrapper.properties`, `gradlew`,
`gradlew.bat`. Exactly the design's slice-2 file list. `git diff 6c22c77..HEAD -- .gitignore | wc -l`
→ `0` (`.gitignore` unchanged since slice 1). `git ls-files build .gradle | wc -l` → `0`;
`git ls-files -ci --exclude-standard | wc -l` → `0`.

**Commit:** first line `gradle-upgrade slice 2: Gradle 9.7.1 wrapper, build script fixes, gate green on JDK 21`
(`git log -1 --format=%s`), the design's slice-2 title verbatim; trailer
`Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` (`git log -1 --format='%(trailers:key=Co-Authored-By)'`).

**`build.gradle` and `gradle.properties` against § Interfaces:**
`sed -n '/^```groovy$/,/^```$/p' <design> | sed '1d;$d' | diff - build.gradle && echo BUILD-IDENTICAL`
→ `BUILD-IDENTICAL`; the same with `properties` and `gradle.properties` → `PROPS-IDENTICAL`
(the design holds exactly one fence of each kind: `grep -c` → `1`, `1`). `wc -l` → `32`, `2`;
`tail -c1 … | xxd` → `0a` for both. `gradle.properties` holds one non-comment key
(`grep -v '^#' gradle.properties | grep -v '^$'` → `org.gradle.java.installations.auto-download=false`),
nothing beyond the design. The design's line map holds: `sed -n '8,12p'` is the `java { toolchain …}`
block, `18,22p` the `dependencies` block with the launcher line, `24,26p` `test { useJUnitPlatform() }`,
`28,32p` the `jar { manifest …}` block, line `30` is `attributes "Main-Class": "Main.Main"`.

**Settled decisions:** 1 (9.7.1: `--version`, `distributionUrl`), 2 (toolchain 21 and
`auto-download=false`: `javaToolchains`, `javap`, no `~/.gradle/jdks`), 3 (`Main-Class: Main.Main`
in the manifest; no `compile` block; no `application`/`run`), 5 (api `5.6.0` kept, engine and
launcher versionless, aligned by the BOM; no bump), 6 (`codegraph.toml`, `scip-java.json` in the
empty base diff). Decision 4 was slice 1 and is intact (above). No decision was reinterpreted and
the slice met no unsettled one.

**Wrapper files** (`cat -n gradle/wrapper/gradle-wrapper.properties`): nine keys, the five original
plus `networkTimeout=10000`, `retries=0`, `retryBackOffMs=500`, `validateDistributionUrl=true`;
line 3 is exactly `distributionUrl=https\://services.gradle.org/distributions/gradle-9.7.1-bin.zip`;
no `distributionSha256Sum`. `ls -l gradle/wrapper/gradle-wrapper.jar` → `47505` bytes.
`git ls-files -s gradlew` → `100755 249efbb0…`; `gradlew.bat`, the jar, the properties,
`build.gradle`, `gradle.properties` → `100644`. `shasum gradlew gradlew.bat gradle/wrapper/*` →
`d736ad5d…`, `1d54ad9d…`, `492e069f…`, `dce1e319…`: the same four digests the builder's evidence
records, so the committed files are the ones the builder's three `wrapper` runs produced.

**Are they the 9.7.1 `wrapper` task's own output?**
`JAVA_HOME=$J21 ./gradlew wrapper --gradle-version 9.7.1 --rerun-tasks --warning-mode all; echo "exit=$?"`
→ `> Task :wrapper`, `BUILD SUCCESSFUL in 783ms`, `1 actionable task: 1 executed`, `exit=0`, no
deprecation line; `diff` of `shasum` before/after → `SHA-IDENTICAL`; `git status --porcelain` →
empty. (With the new `build.gradle` this run prints no "Deprecated Gradle features" summary; the
builder's bootstrap runs printed it because the old script was still in place, as the evidence says.)
Read-only comparison with the Lead's stash (`git show 'stash@{0}':gradle/wrapper/gradle-wrapper.jar | cmp - gradle/wrapper/gradle-wrapper.jar`)
→ `JAR-IDENTICAL-TO-STASH`; the properties file is identical to the stash's too. The jar's own
manifest (`unzip -p … META-INF/MANIFEST.MF`) names `Main-Class: org.gradle.wrapper.GradleWrapperMain`,
`Implementation-Title: Gradle Wrapper`.

**Test dependency resolution:** as the design expects (§ Interfaces, "Test dependency resolution"):
engine `5.6.0`, launcher `1.6.0`, no `FAILED`. **Download:** the launcher is in the cache at
`~/.gradle/caches/modules-2/files-2.1/org.junit.platform/junit-platform-launcher/1.6.0/` (`.module`,
`.jar`, `.pom`), all three with mtime `Oct 16 17:03` (`ls -l`), i.e. the builder's run; none of my
runs added a file there or a directory under `org.junit.platform/` (still exactly `junit-platform-commons`,
`junit-platform-engine`, `junit-platform-launcher`). No download by the verifier.

**Builder's evidence, slice-2 section, every number reproduced:** wrapper digests (4/4), jar `47505`,
properties keys (9), `BUILD-IDENTICAL`/`PROPS-IDENTICAL`, `32`/`2` lines, dependency tree (same
versions, `FAILED` count `0`), `Gradle 9.7.1` / `21.0.12.1`, `1w1c7tv4s851m17nbqdsro2tv`,
`Auto-download: Disabled`, gate `exit=0` with 7/6/1 tests and `0/0/0`, `major version: 65` ×2,
`build` `exit=0`, manifest, `6` classes with the listed sizes, `totalProblemCount":2`, no
`~/.gradle/jdks`, `gradlew` mode `100755`, empty frozen-file diff. The only part of the builder's
evidence that I could not reproduce as described is the JDK 27 attempt, because it is reproducible
only in its inconclusiveness; see the probe below.

## Adversarial probes

1. **Stale XML / hidden deprecations.** The gate was run with `clean` first and without `-q`; the
   XML timestamps are from the run (`14:07:10Z`, machine time `17:07`), not `2021-03-19`. Both
   `--warning-mode all` runs and the `--info` run carry no `deprecat`/`warning`/`Deprecated Gradle
   features` line (`grep -l 'Deprecated Gradle features' *.log` over all my logs → nothing, except
   the `-P` run in probe 2, which is a command-line deprecation, not a script one).

2. **JDK 27, the real attempt.** The builder's attempt (`JAVA_HOME=$J27 ./gradlew test --warning-mode all -Porg.gradle.java.installations.paths=$J21`,
   no `clean`) reported every task `UP-TO-DATE`, so it only showed that Gradle 9.7.1 starts on 27;
   the builder's evidence says exactly that ("the tests were **not re-executed**") and does not claim
   more, so it does not overstate; but it leaves acceptance criterion 4 ("attempted once and recorded")
   without a test result, and the commit message's "gate green on JDK 21" is correct only because it
   says nothing about 27. Three runs by me, each with `clean`:

   **(A)** `JAVA_HOME=$J27 ./gradlew clean test --warning-mode all -Dorg.gradle.java.installations.paths=$J21; echo "exit=$?"`
   → `exit=0`; `> Task :clean`, `:compileJava` (the same two javac notes), `:compileTestJava`,
   `:test` all executed, `BUILD SUCCESSFUL in 1s`, `4 actionable tasks: 4 executed`; no
   `deprecat`/`warning` line (`grep exit=1`). Fresh XML: `tests=14 failures=0 errors=0 skipped=0`,
   timestamps `2026-10-16T14:08:59.557Z`/`…567Z`/`…572Z`. `javap` after this run and before any
   JDK 21 run: `major version: 65` for `Main.Main` and `TodoList.TaskTest`; `xxd` → `0041` for all 9
   class files. `JAVA_HOME=$J27 ./gradlew -q javaToolchains -D…` lists `Homebrew JDK 21 (21.0.12.1)`
   `Detected by: Gradle property 'org.gradle.java.installations.paths'` and `Homebrew JDK 27 (27)`
   `Detected by: Current JVM`, `Auto-download: Disabled`. A re-run with `--info` shows which JVM did
   the work: `Compiling with toolchain '/opt/homebrew/Cellar/openjdk@21/21.0.12.1/libexec/openjdk.jdk/Contents/Home'`
   and `Starting process 'Gradle Test Executor 5'. … Command: /opt/homebrew/Cellar/openjdk@21/21.0.12.1/libexec/openjdk.jdk/Contents/Home/bin/java …`
   while the daemon's `jvmCriteria` is `/opt/homebrew/Cellar/openjdk/27/…`. **So: Gradle 9.7.1 runs on
   JDK 27 and, with JDK 21 made discoverable, the whole gate is green on the Java 21 toolchain.**

   **(B)** `JAVA_HOME=$J27 ./gradlew clean test --warning-mode all; echo "exit=$?"` (no installations
   path) → `exit=1`, `BUILD FAILED in 240ms`, verbatim:

   ```
   * What went wrong:
   Could not determine the dependencies of task ':test'.
   > Could not resolve all dependencies for configuration ':testRuntimeClasspath'.
      > Failed to calculate the value of task ':compileTestJava' property 'javaCompiler'.
         > Cannot find a Java installation on your machine (Mac OS X 27.0.1 aarch64) matching: {languageVersion=21, vendor=any vendor, implementation=vendor-specific, nativeImageCapable=false}. Toolchain auto-provisioning is not enabled.
   ```

   `javaToolchains` on bare 27 lists only `Homebrew JDK 27 (27)`; `/usr/libexec/java_home` (which the
   `--info` log shows Gradle invoking for auto-detection) does not report the Homebrew JDK 21, so the
   installed JDK 21 is invisible unless it is the current JVM or named by a property. The failure
   is the designed behaviour (decision 2: no auto-provisioning) and the message is clear. No
   `~/.gradle/jdks` appeared.

   **(C)** The design's exact `-P` form, after `clean`:
   `JAVA_HOME=$J27 ./gradlew clean test --warning-mode all -Porg.gradle.java.installations.paths=$J21`
   → `exit=0`, all four tasks executed, fresh XML `1/6/7` tests `0/0/0`, timestamps `14:09:01Z`,
   preceded by the deprecation line
   `Specifying 'org.gradle.java.installations.paths' as a project property on the command line has been deprecated. This is scheduled to be removed in Gradle 10. Instead, specify it as a Gradle property: '-Dorg.gradle.java.installations.paths=…'.`
   So `-P` still works in 9.7.1, with a warning that comes from the command line, not `build.gradle`.

   **Was the design's JDK 27 command adequate?** No, on two counts, neither blocking: it lacks
   `clean` (or `--rerun-tasks`), so run after the JDK 21 gate it is a no-op because the toolchain
   makes the compile/test inputs identical on both launcher JDKs; and it uses the `-P` form, which
   9.7.1 deprecates in favour of `-D`. The recorded result the docs may use is (A) and (B). Risk R10
   (property rejected) did not occur; risk R9 (Gradle does not run on 27) did not occur.

   After the JDK 27 runs the JDK 21 gate was re-run (§ Mechanical gate, second run) and
   `git status --porcelain` is empty.

3. **Behaviour silently changed versus base** (`git show $BASE:build.gradle`): the removed
   `tasks.withType(Test) { useJUnitPlatform(); jvmArgs '--enable-preview' }` and the removed
   `from { configurations.compile… }` block. `/usr/bin/grep -rn 'preview' src/ build.gradle` → nothing;
   the gate is green without the flag on 21 (and on the 27-launched run), so the flag was dead. One
   correction to the contract: `/usr/bin/grep -rnE 'case [^:]*->' src/` finds **14 arrow-switch
   cases** (`TodoList.java:118-151`, `UserInterface.java:46-100`), which the contract said were absent
   (contract § What changes, `build.gradle:23` row). Arrow `switch` has been a standard feature since
   Java 14, so the conclusion (no preview feature, flag unused) still holds; the fact in the
   contract is wrong and slice 3 should not repeat it. `./gradlew build` still produces
   `build/libs/todolist-1.0-SNAPSHOT.jar` with the six project classes and `Main-Class: Main.Main`,
   and nothing from a dependency (there are none on the main classpath); starting it was not tried.

4. **Toolchain honoured when Gradle runs on 27:** probe 2 (A), `javap` 65 before any JDK 21 run,
   executor command line is the JDK 21 `java`.

5. **JDK download:** `ls ~/.gradle/jdks` → `No such file or directory` before and after all runs;
   no `Provisioning`/`download` line in any log; `Auto-download: Disabled` on 21 and 27.

6. **`gradle.properties` beyond the single key:** one key only (above); no user-level
   `~/.gradle/gradle.properties` or `init.d` exists, so nothing else configures the build.

7. **`gradlew` script:** `head -1 gradlew` → `#!/bin/sh`; `file gradlew` → `POSIX shell script text
   executable, … UTF-8 text`; `grep -c $'\r' gradlew` → `0`; `/bin/sh -n gradlew` → `SH-SYNTAX-OK`;
   `grep -n 'gradle-wrapper.jar' gradlew` → `213: -jar "$APP_HOME/gradle/wrapper/gradle-wrapper.jar"`
   (the 9.x launch form). `gradlew.bat` has `82` CR lines (`file` → `DOS batch file … CRLF`), as a
   batch file should and as the base's `gradlew.bat` had (`89`); `build.gradle`,
   `gradle.properties`, `gradle-wrapper.properties` → `0` CR.

8. **Problems report:** the gate's `build/reports/problems/problems-report.html` (copied before
   `clean`) holds `"totalProblemCount":2`; parsed with `python3 -I`, both entries have
   `problemId` `compilation > java > compiler.note.unchecked.filename` / `…unchecked.recompile`,
   `severity: WARNING`, `locations: [{path: …/src/main/java/TodoList/FileReader.java}, {taskPath: ":compileJava"}]`;
   the string `build.gradle` occurs `0` times and `deprecat` `0` times in the file. Neither entry is
   attributable to `build.gradle`; they are javac's unchecked notes on frozen source, unchanged
   from the base (the contract's SER-1 rule forbids silencing them in `src/`).

9. **Wrapper files not the task's own output:** probe above (`--rerun-tasks` → `SHA-IDENTICAL`,
   clean status; identical to the stash's jar and properties).

10. **Leftover `compile`/`--enable-preview`/`FizzBuzz`/second `useJUnitPlatform()`:** the `grep`
    in § Invariants → only `:25`.

11. **Revert leaves slice 1 intact:** `git revert --no-commit HEAD` → `exit=0`; during the revert
    `git ls-files build .gradle | wc -l` → `0`, `git diff 6c22c77 -- .gitignore | wc -l` → `0`,
    `git diff --cached $BASE -- build.gradle gradle/wrapper/* gradlew gradlew.bat | wc -l` → `0`
    (the six build files return to base), `gradle.properties removed`; `git revert --abort` →
    `exit=0`, `git status --porcelain` empty, HEAD still `60835d1…`, `git diff --quiet HEAD -- build.gradle gradle.properties gradlew gradlew.bat gradle/` → `UNCHANGED`.

12. **`settings.gradle` on 9.7.1:** unchanged (`rootProject.name = 'todolist'`, in the empty frozen
    diff) and the build configures without complaint (R11 did not occur).

## Verdict

**green** — the whole mechanical gate reproduces on JDK 21 (`exit=0`, 14 tests, 0 failures, 0
errors, 0 skipped, no deprecation or warning line); `build.gradle` and `gradle.properties` are
byte-identical to the design; the wrapper files are the 9.7.1 `wrapper` task's own stable output
(47505-byte jar, `100755` script, nine property keys, no checksum); `build` exits 0 with
`Main-Class: Main.Main` and six project classes; the toolchain compiles for 21 (major 65) even
with Gradle launched on JDK 27; no JDK was downloaded; every frozen file and `.gitignore` are
unchanged; the worktree is clean after every run; every number in the builder's slice-2 evidence
reproduces; the commit's first line is exact. No unsettled decision was touched.

Two things slice 3 must state honestly, neither a blocker for slice 2:

- **JDK 27 result for `BUILD-1`:** the builder's attempt executed no tests (its evidence says so).
  The observed result is: Gradle 9.7.1 starts and runs on JDK 27; with
  `-Dorg.gradle.java.installations.paths=<JDK 21>` the gate is green (14/0/0/0) on the Java 21
  toolchain; with bare `JAVA_HOME=<JDK 27>` it fails with "Cannot find a Java installation on your
  machine … matching: {languageVersion=21 …}. Toolchain auto-provisioning is not enabled." because
  the Homebrew JDK 21 is not auto-detected (`/usr/libexec/java_home` does not list it). The design's
  phrase "a JDK 21 must be installed" is therefore not enough on this machine: it must be the
  current JVM (`JAVA_HOME`) or named by `org.gradle.java.installations.paths`. The design's JDK 27
  command (`-P`, no `clean`) was inadequate: `-P` is deprecated in 9.7.1 in favour of `-D`, and
  without `clean` the run is a no-op after a JDK 21 gate.
- **Contract inaccuracy:** `src/` does use arrow `switch` (14 cases); the `--enable-preview`
  removal is still correct because arrow `switch` is standard since Java 14, as the green gate
  without the flag shows.
