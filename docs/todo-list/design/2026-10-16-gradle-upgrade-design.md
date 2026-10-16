# Design — gradle-upgrade

> Consumes the approved contract at `docs/todo-list/design/2026-10-16-gradle-upgrade-contract.md`
> (cited below as "contract:<line>"). Does not re-derive its facts.
> Architect: `architect-gradle-upgrade`, roster `strict`, 2026-10-16. Repository `todo-list-java`,
> base `main` @ `46cd9856e9d589a2ce56ad2006bb20b7599372da`.
>
> **Citations.** Repository facts come from the contract, cited as it cites them. Where this design
> needed a fact the contract does not state, it is marked **(architect's read)** with the
> command or `path:line` used. Nothing here was obtained by running `./gradlew` or the application.
> Statements about Gradle behaviour that nobody has observed on this machine stay marked **[verify]**
> and are assigned to the slice that confirms them (§ Risks).

## Settled decisions (Contract gate, 2026-10-16, human)

Fixed. A builder or verifier who finds one of them infeasible stops and returns `blocked`; nobody
reinterprets them.

1. **Target Gradle 9.7.1** (latest 9.x), already cached in `~/.gradle/wrapper/dists` (contract:141-143).
2. **Pin a Java 21 toolchain** in `build.gradle`: `java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }`.
   **No JDK auto-provisioning**: nothing may download a JDK. Made explicit by a new
   `gradle.properties` with `org.gradle.java.installations.auto-download=false` (§ Interfaces).
3. **Jar `Main-Class` becomes `Main.Main`**; the `configurations.compile` fat-jar block
   (`build.gradle:31-33`, contract:178) is removed. **No `application` plugin, no `run` task.**
   Consequence: IO-1 and UI-1 stay out of scope (contract:111-118).
4. **Untrack `build/` and `.gradle/`** (`git rm --cached`, 25 + 30 files, contract:222-224) and
   replace the malformed `.gitignore:1-3` (contract:225-229) with `build/`, `.gradle/`, `.DS_Store`,
   `bin/`, `out/`. `.idea/` is left alone (tracked, unchanged, including its JDK 15 settings,
   contract:230-233).
5. **JUnit:** keep `junit-jupiter-api:5.6.0` and the versionless engine; **add
   `junit-platform-launcher` versionless** so the JUnit BOM aligns it. No JUnit bump.
6. **Codegraph `build_tool` switch deferred** to a follow-up. `codegraph.toml` and
   `scip-java.json` stay byte-identical; the pending note at `codegraph.toml:19-21` stays accurate.

**Scope of change.** Unfrozen for this feature only: `build.gradle`, `gradle/wrapper/*`, `gradlew`,
`gradlew.bat`, `.gitignore`, and the tracked `build/` and `.gradle/` files (index removal only).
New: `gradle.properties`. Docs: `AGENTS.md`, `readme.md`, `docs/todo-list/README.md`, one evidence
file. **Must not change:** `settings.gradle`, `objectFile.txt`, `dataFile.txt`, `codegraph.toml`,
`scip-java.json`, `.idea/**`, and everything under `src/` (empty diff).

## Shape of the change

Three things change, in this order, each one slice:

```
slice 1  build state out of git      .gitignore rewritten; build/ and .gradle/ removed from the index
           │                          (gate runs from here on leave `git status` clean)
           ▼
slice 2  Gradle 6.7 -> 9.7.1          wrapper bootstrapped and regenerated; build.gradle fixed;
         green on JDK 21              gradle.properties added; gate green on 21; JDK 27 attempted once
           │
           ▼
slice 3  docs say what was observed   AGENTS.md, readme.md, evidence "Not verified here", README index
```

**Why untracking goes first** (deviation from the Lead's suggested order, which put it second): the
stashed experiment shows that any 9.7.1 run rewrites tracked files under `build/` and `.gradle/`
(contract:208-218). With those still tracked, the slice-2 builder would have to hand-pick its commit
out of dozens of build-output diffs, and the verifier could not use "`git status --porcelain` is
empty after the gate" as a check that the gate touched nothing it should not (`objectFile.txt`,
IO-1, contract:111-115). After slice 1 that check is one command. Slice 1 is verifiable without
the gate (it is pure index and ignore-rule work), and it is revertible on its own.

**Why the wrapper and the build script are one slice:** on JDK 21 neither half is green without the
other. The 6.7 wrapper cannot start on 21 (contract:184-187); the 9.7.1 wrapper with the old script
lacks the launcher (contract:174) and keeps the `compile` reference that breaks `./gradlew build`
(contract:178). The only green state is both together. Reviewers can still separate them by file:
the four wrapper files are generated, `build.gradle` and `gradle.properties` are hand-written.

### Precondition (Lead, before slice 1)

The contract, this design and the `docs/todo-list/README.md` lines listing them exist only in the
primary checkout's working tree today (architect's read: `git status --short` shows
` M docs/todo-list/README.md`, `?? docs/todo-list/design/`). The worktree is created from the base
SHA, which has neither. The Lead commits them onto `feature/gradle-upgrade` as
`gradle-upgrade: contract and design` before spawning the slice-1 builder, so every slice's README
edit lands on top of them. The slice-1 builder checks that
`docs/todo-list/design/2026-10-16-gradle-upgrade-design.md` exists in the worktree and returns
`blocked` if it does not.

### Common environment for every command in this design

Run from the worktree root `.claude/worktrees/gradle-upgrade`.

```
J21=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home
J27=/opt/homebrew/opt/openjdk@27/libexec/openjdk.jdk/Contents/Home
BASE=46cd9856e9d589a2ce56ad2006bb20b7599372da
```

Every `./gradlew` invocation is prefixed `JAVA_HOME=$J21` (or `$J27` for the one attempt), never a
bare `java` from `PATH`: `/opt/homebrew/opt/openjdk` is JDK 27 (contract:204-206).

## Interfaces

Configuration only; no Java signature, schema or runtime data changes (contract:135-136).

### `gradle/wrapper/gradle-wrapper.properties`

- Bootstrap edit (hand, slice 2 step 1): line 3 becomes exactly
  `distributionUrl=https\://services.gradle.org/distributions/gradle-9.7.1-bin.zip`
  (today `gradle-6.7-bin.zip`, contract:139-140; architect's read of
  `gradle/wrapper/gradle-wrapper.properties:1-5`). Exact spelling matters: the wrapper names its
  cache directory after a hash of this URL, and only this spelling hits the cached distribution
  (architect's read: `~/.gradle/wrapper/dists/gradle-9.7.1-bin/` holds one directory,
  `1w1c7tv4s851m17nbqdsro2tv`).
- Final content is whatever the 9.7.1 `wrapper` task writes. Expected shape per the stash
  (contract:85-94): the five existing keys plus `networkTimeout=10000`, `retries=0`,
  `retryBackOffMs=500`, `validateDistributionUrl=true`. **[verify, slice 2]**
- **No `distributionSha256Sum`.** Decided here: the checksum could only be taken from the locally
  cached zip, not confirmed against Gradle's published value without a network fetch this feature
  does not need, so a pin would add no assurance. A follow-up can add it with
  `--gradle-distribution-sha256-sum`.

### `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar`

Generated by `./gradlew wrapper --gradle-version 9.7.1 --distribution-type bin` on JDK 21, run twice
(second run proves the files are the 9.7.1 task's own output and stable). Never copied from the
stash, never hand-edited (contract:85-94). `gradlew` keeps mode `100755` in the index.

### `build.gradle` — exact target content (32 lines)

```groovy
plugins {
    id 'java'
}

group = 'org.example'
version = '1.0-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.6.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

test {
    useJUnitPlatform()
}

jar {
    manifest {
        attributes "Main-Class": "Main.Main"
    }
}
```

Line map (the docs in slice 3 cite these): toolchain `:8-12`, dependencies `:18-22`, test task
`:24-26`, jar manifest `:28-32`, `Main-Class` `:30`. The file ends with a single newline.

Construct-by-construct, against the contract's table (contract:167-180):

| Contract row | Decision | Reason |
|---|---|---|
| `:1-3` plugins | unchanged | current form |
| `:5-6` space-assignment | `group = …`, `version = …` | observed 9.7.1 deprecation at these two lines (contract:170) |
| (new) toolchain | added at `:8-12` | settled decision 2 |
| `:8-10` repositories | unchanged | current |
| `:13-14` JUnit | unchanged | settled decision 5 |
| (new) launcher | `testRuntimeOnly 'org.junit.platform:junit-platform-launcher'`, versionless | settled decision 5; contract:174 |
| `:17-19` `test { useJUnitPlatform() }` | kept, the single `useJUnitPlatform()` | contract:175 asks for exactly one |
| `:21-24` `tasks.withType(Test) { … }` | **block deleted** | its two lines are the duplicate `useJUnitPlatform()` and `--enable-preview`, which nothing uses (contract:176); deleting it removes the duplicate and the dead flag, and spares verifying preview-flag behaviour on two JDKs |
| `:26-29` `Main-Class` | value `Main.Main` | settled decision 3 |
| `:31-33` `configurations.compile` | **deleted**, not replaced | settled decision 3; no main dependencies exist, so the block was a no-op (contract:178) |
| no `sourceCompatibility` | none added | the toolchain sets the level; adding both is redundant |

Not added: `application` plugin, `mainClass`, `run`, Kotlin DSL, any other plugin, any
`junit-bom` platform, any toolchain resolver plugin (that would need `settings.gradle`, which is
frozen).

### `gradle.properties` — new file, exact content

```properties
# The Java 21 toolchain in build.gradle must be an installed JDK; Gradle never downloads one.
org.gradle.java.installations.auto-download=false
```

There is no `gradle.properties` today (architect's read: root listing) and no user-level
`~/.gradle/gradle.properties` on this machine that could override it (architect's read: `ls`). Two
independent locks keep a JDK from downloading: this property, and the absence of any toolchain
download repository (Gradle 8+ provisions only through a resolver plugin declared in
`settings.gradle`, which is frozen and declares none, `settings.gradle:1`) **[verify, slice 2]**.
The verifier observes the property taking effect with `./gradlew -q javaToolchains`.

No machine path goes into `gradle.properties`. For the single JDK 27 attempt, the JDK 21 location is
passed on the command line instead (§ Slices, slice 2).

### `.gitignore` — exact content (replaces all three lines)

```
build/
.gradle/
.DS_Store
bin/
out/
```

Rules the verifier checks: `gradle/` (the wrapper directory), `gradlew`, `gradlew.bat`,
`gradle.properties`, `.idea/**`, `src/**`, `objectFile.txt`, `dataFile.txt` are **not** ignored;
no tracked file outside `build/` and `.gradle/` becomes ignored (`git ls-files -ci --exclude-standard`
is empty). Architect's read: `git ls-files | grep -iE '(^|/)(build|\.gradle|bin|out)/|\.DS_Store'`
matches nothing outside `build/` and `.gradle/`, so the patterns catch nothing else that is tracked.

### Test dependency resolution (expected, **[verify, slice 2]**)

`./gradlew dependencies --configuration testRuntimeClasspath` shows `junit-jupiter-engine`,
`junit-platform-launcher`, `junit-platform-engine` and `junit-platform-commons` aligned by
`org.junit:junit-bom:5.6.0` (engine `5.6.0`, platform artifacts `1.6.0`), with no `FAILED` entry.

**Download note.** `junit-platform-launcher` is **not** in the local Gradle cache (architect's read:
`~/.gradle/caches/modules-2/files-2.1/org.junit.platform/` holds only `junit-platform-commons` and
`junit-platform-engine`). Resolving it is expected to download it from Maven Central. That is allowed
(dependency, not JDK); the builder records it (§ Slices, slice 2 step 5).

### Frozen, public, unchanged

`settings.gradle`, `objectFile.txt`, `dataFile.txt`, `codegraph.toml`, `scip-java.json`, `.idea/**`,
`src/**`. The mechanical gate command stays `./gradlew test`. The jar name stays
`build/libs/todolist-1.0-SNAPSHOT.jar` (root project `todolist`, `settings.gradle:1`, version
`1.0-SNAPSHOT`).

## Slices, in order

Roster `strict`: no slice is helper-eligible. One commit per slice, named
`gradle-upgrade slice N: <what>`. Every slice appends its own section to the evidence file (below)
in the same commit. The verifier runs the mechanical gate itself at every slice and records the
result; the expected gate result is stated per slice.

1. **slice-1 — untrack build state and fix `.gitignore`** — `git ls-files build .gradle` is empty,
   the five patterns ignore what they should and nothing else, the working tree still holds the files.
2. **slice-2 — Gradle 9.7.1 wrapper, build script fixes, gate green on JDK 21** — on JDK 21
   `./gradlew test --warning-mode all` exits 0 with 14 tests, 0 failures, 0 errors, 0 skipped and no
   deprecation lines; `./gradlew build` exits 0; manifest says `Main.Main`; JDK 27 attempted once.
3. **slice-3 — docs state what was observed** — `AGENTS.md` and `readme.md` contain no Gradle 6.7 /
   JDK 15 claim and every new claim matches a line of the evidence file; `bin/check` heading checks pass.

### Slice 1 — untrack build state and fix `.gitignore`

**Files:** `.gitignore` (rewritten), `build/**` and `.gradle/**` (index removal, 55 paths),
`docs/todo-list/evidence/2026-10-16-gradle-upgrade.md` (created), `docs/todo-list/README.md`
(evidence list gains the evidence file).

**Builder steps:**

1. `git rm -r --cached --quiet build .gradle`
2. Write `.gitignore` with exactly the five lines in § Interfaces.
3. Baseline, recorded not fixed: `JAVA_HOME=$J21 ./gradlew test` — expected to fail at startup with
   `Unsupported class file major version 65` (contract:184-187). This documents the before-state on
   the branch; whatever it writes now lands in ignored directories.
4. Create the evidence file (preamble + `## Slice 1` section), add it to the README evidence list.

**Verification (builder records, verifier re-runs):**

| Command | Expected observed outcome |
|---|---|
| `git ls-files build .gradle \| wc -l` | `0` |
| `git diff --name-status HEAD~1 HEAD \| grep -vE '^D\s+(build\|\.gradle)/'` | `.gitignore` (M), the evidence file (A), `docs/todo-list/README.md` (M); nothing else |
| `git diff --name-status HEAD~1 HEAD \| grep -cE '^D\s+(build\|\.gradle)/'` | `55` (25 + 30, contract:222-224) |
| `git check-ignore -v build/x .gradle/x bin/x out/x .DS_Store src/main/.DS_Store` | each path reported against `.gitignore:<n>` |
| `git check-ignore gradle/wrapper/gradle-wrapper.jar gradlew gradlew.bat gradle.properties .idea/misc.xml src/main/java/Main/Main.java objectFile.txt dataFile.txt; echo $?` | no path printed, `1` |
| `git ls-files -ci --exclude-standard` | empty |
| `test -d build && test -d .gradle && echo present` | `present` (only the index changed) |
| `git diff $BASE..HEAD -- src/ objectFile.txt dataFile.txt settings.gradle build.gradle gradle/ gradlew gradlew.bat codegraph.toml scip-java.json .idea/` | empty |
| mechanical gate `JAVA_HOME=$J21 ./gradlew test` | **red**, `Unsupported class file major version 65` (same as base; slice 1 does not touch the build) |
| `git status --porcelain` after the gate | empty |

**The verifier tries to break it:** a pattern that is too wide (does `build/` or `bin/` hit anything
tracked anywhere in the tree, does `.gradle/` hit `gradle/`); a tracked file outside `build/` and
`.gradle/` removed from the index; files deleted from disk instead of only the index; a `.gitignore`
with stray text, CRLF or a leading `/` that changes matching; `.idea/` touched; a revert
(`git revert --no-commit HEAD && git ls-files build .gradle | wc -l` → 55, then `git revert --abort`)
restores the index cleanly.

### Slice 2 — Gradle 9.7.1 wrapper, build script fixes, gate green on JDK 21

**Files:** `gradle/wrapper/gradle-wrapper.properties`, `gradle/wrapper/gradle-wrapper.jar`,
`gradlew`, `gradlew.bat` (generated), `build.gradle` (exact content in § Interfaces),
`gradle.properties` (new, exact content in § Interfaces), the evidence file (`## Slice 2`).

**Builder steps, in this order:**

1. Pre-state, recorded: `ls ~/.gradle/wrapper/dists/gradle-9.7.1-bin/*/` and
   `ls ~/.gradle/caches/modules-2/files-2.1/org.junit.platform/`.
2. Bootstrap: set `gradle-wrapper.properties:3` to the exact `distributionUrl` in § Interfaces (the
   only hand edit to a wrapper file).
3. `JAVA_HOME=$J21 ./gradlew wrapper --gradle-version 9.7.1 --distribution-type bin` — the 6.7 launcher
   jar starts the cached 9.7.1 distribution, which regenerates the four wrapper files **[verify]**.
   The old `build.gradle` is still in place here; the stash shows 9.7.1 configures it with only the
   two line-5/6 deprecations (contract:170, :195-199), so this run is expected to succeed with those
   warnings.
4. Run step 3's command again with the new wrapper. Expected: `git status --porcelain gradle gradlew gradlew.bat`
   shows the same four files as after step 3 and `git diff` between the two runs is empty (the
   builder compares `shasum` of the four files before and after the second run).
5. Write `build.gradle` and `gradle.properties` with the exact content in § Interfaces.
6. Dependency resolution, offline first: `JAVA_HOME=$J21 ./gradlew --offline dependencies --configuration testRuntimeClasspath`.
   If it fails only because `junit-platform-launcher` is not cached (expected, § Interfaces), record
   that failure verbatim, then run the same command without `--offline` and record it. The evidence
   states plainly: "Maven Central download: yes/no, artifacts: …" from the two outputs and a second
   `ls` of the cache directory. A failure for any other reason, or any attempt to download a JDK, is
   `blocked`.
7. The gate and the build (table below). Then the JDK 27 attempt, once.

**Verification (builder records, verifier re-runs):**

| Command | Expected observed outcome |
|---|---|
| `JAVA_HOME=$J21 ./gradlew --version` | `Gradle 9.7.1`; `Launcher JVM` / `Daemon JVM` 21.x |
| `grep -n distributionUrl gradle/wrapper/gradle-wrapper.properties` | `:3`, `…/gradle-9.7.1-bin.zip` |
| `ls ~/.gradle/wrapper/dists/gradle-9.7.1-bin/` | still exactly `1w1c7tv4s851m17nbqdsro2tv` (no second download directory) |
| `JAVA_HOME=$J21 ./gradlew -q javaToolchains` | lists a JDK 21 toolchain; auto-download reported disabled **[verify exact wording]** |
| `JAVA_HOME=$J21 ./gradlew clean` then `JAVA_HOME=$J21 ./gradlew test --warning-mode all` | exit `0`, `BUILD SUCCESSFUL`; no line matching `-i 'deprecat'`, no `Deprecated Gradle features were used`, no line naming `build.gradle` |
| `grep -h '<testsuite' build/test-results/test/TEST-*.xml` | three suites: `CompareTasksTest tests="1"`, `TaskTest tests="7"`, `TodoListTest tests="6"`, each `failures="0" errors="0" skipped="0"`, timestamps from the run day, not `2021-03-19` (contract:68-72) |
| `JAVA_HOME=$J21 ./gradlew dependencies --configuration testRuntimeClasspath` | engine `5.6.0`, launcher `1.6.0` via the BOM, no `FAILED` |
| `$J21/bin/javap -v -cp build/classes/java/main Main.Main \| grep 'major version'` | `major version: 65` (toolchain 21) |
| `JAVA_HOME=$J21 ./gradlew build --warning-mode all` | exit `0`, no deprecation lines |
| `unzip -p build/libs/todolist-1.0-SNAPSHOT.jar META-INF/MANIFEST.MF` | contains `Main-Class: Main.Main` |
| `unzip -l build/libs/todolist-1.0-SNAPSHOT.jar \| grep -c '\.class$'` | `6`, the main classes under `Main/` and `TodoList/` (the six the stash compiled, contract:210-211); nothing unpacked from a dependency |
| `git status --porcelain` after all runs | empty (build output ignored; `objectFile.txt` untouched, IO-1) |
| `git diff --name-only HEAD~1 HEAD` | exactly the four wrapper files, `build.gradle`, `gradle.properties`, the evidence file |
| `git ls-files -s gradlew` | mode `100755` |
| `git diff $BASE..HEAD -- src/ objectFile.txt dataFile.txt settings.gradle codegraph.toml scip-java.json .idea/` | empty |
| JDK 27 attempt, once: `JAVA_HOME=$J27 ./gradlew test --warning-mode all -Porg.gradle.java.installations.paths=$J21` | **recorded, not required green.** Either Gradle 9.7.1 refuses to start on 27, or it starts and runs the tests on the Java 21 toolchain. The `-P` makes the attempt answer "does Gradle run on 27" rather than "does Gradle find a JDK 21"; if Gradle rejects the property, the verbatim error is the result **[verify]** |

**Mechanical gate expected:** green on JDK 21 (exit 0, 14/0/0/0).

**The verifier tries to break it:** counts taken from stale XML (run `clean` first, check
timestamps); deprecations hidden by `-q` or missing `--warning-mode all`; the toolchain actually not
honoured (run Gradle on JDK 27 and check the test classes' major version is still 65 if that run gets
that far); a JDK download (check `~/.gradle/jdks/` does not appear or gain an entry, and grep the
output for `Provisioning`/`download` of a JDK); wrapper files that are not the task's own output
(re-run `./gradlew wrapper --gradle-version 9.7.1 --distribution-type bin` and expect no diff; compare
`gradle-wrapper.jar` with the stash's copy, `cmp <(git show 'stash@{0}':gradle/wrapper/gradle-wrapper.jar) gradle/wrapper/gradle-wrapper.jar`,
read-only, expected identical and 47505 bytes (contract:92) — a mismatch is reported and explained,
not blocking on its own when the re-run is clean); a leftover reference to `compile`,
`--enable-preview`, `FizzBuzz` or a second `useJUnitPlatform()` (`grep -nE 'compile|enable-preview|FizzBuzz|useJUnitPlatform' build.gradle`
→ only `:25`); `build.gradle` differing from the exact content in § Interfaces (`diff` against it);
a `src/` change made to silence a javac 21 warning (SER-1, contract:105-110); the slice reverted on
its own (`git revert --no-commit HEAD`) leaves slice 1 intact.

### Slice 3 — docs state what was observed

**Files:** `AGENTS.md`, `readme.md`, the evidence file (final `## Slice 3` and `## Not verified here`),
`docs/todo-list/README.md` (only if a line needs updating; verify reports are listed by verifiers).

Every number, version and JDK named in these docs must appear in a command output in the evidence
file. Where the text below has `<…>`, the builder fills it from the evidence file and nothing else.

**`AGENTS.md` edits** (line numbers at base, contract:237-246, architect's read of `AGENTS.md:1-71`):

| Line | Today | Becomes |
|---|---|---|
| `:13-14` | "Gradle 6.7 wrapper build, JUnit 5 tests, no CI." | "Gradle 9.7.1 wrapper build with a Java 21 toolchain, JUnit 5 tests, no CI." |
| `:26` | "Gradle 6.7 build of root project `todolist`" | row path becomes `` `build.gradle`, `settings.gradle`, `gradle.properties`, `gradle/wrapper/` ``; text "Gradle 9.7.1 build of root project `todolist`, Java 21 toolchain, no JDK auto-download" |
| `:38` BUILD-1 | "The wrapper pins Gradle 6.7, which runs only on JDK 15 or older; …" | "`BUILD-1` The wrapper pins Gradle 9.7.1 (`gradle/wrapper/gradle-wrapper.properties:3`) and `build.gradle:8-12` pins a Java 21 toolchain; `gradle.properties` disables JDK auto-download, so a JDK 21 must be installed. The gate is observed green with JAVA_HOME = JDK 21. With JAVA_HOME = JDK 27: <one clause, the observed result>." No JDK range beyond what was observed. |
| `:43` gate comment | "(build.gradle:16-22). Needs JAVA_HOME = a JDK ≤ 15 (BUILD-1)." | "`./gradlew test        # JUnit 5 via the wrapper (build.gradle:18-26). Needs JAVA_HOME = a JDK 21 (BUILD-1).`" |
| `:46` | "the readme's `gradle test` is the only documented check" | "`./gradlew test` (also in the readme) is the only documented check" |
| `:52-54` Branching frozen list | `build.gradle`, `settings.gradle`, `gradle/wrapper/`, … | add `gradle.properties` after `settings.gradle` (new build configuration file, Lead-owned like its neighbours) |
| `:59` | "export JAVA_HOME=<JDK 15 or older>      # Gradle 6.7 limit (BUILD-1)" | "export JAVA_HOME=<a JDK 21>             # e.g. /opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home (BUILD-1)" |
| `:60` | "# compiles into build/ (committed, see .gitignore)" | "# compiles into build/ (ignored, see .gitignore)" |
| `:62` | `java -cp build/classes/java/main Main.Main` | unchanged |
| `:65-67` | "The readme's `gradle run` has no task behind it … `com.gradle.tutorial.FizzBuzzProcessor` … does not start the app." | "There is no `run` task: `build.gradle` applies only the `java` plugin, not `application`; start the app with the `java -cp` line above. The jar manifest names `Main.Main` as `Main-Class` (`build.gradle:30`); starting the app from the jar is not verified by agents." |

`AGENTS.md:1` (the `@../dev-agent-kit/WORKSPACE.md` import) and the four required headings stay
exactly as they are.

**`readme.md` edits** (architect's read of `readme.md:6-32`):

- `:6-10` "Build with": `- Gradle 9.7.1 (wrapper, \`./gradlew\`)`, `- JDK 21`, `- JUnit 5`.
- `:18-32` "Run with Gradle" becomes "Build and run": go to the root folder; `export JAVA_HOME=<a JDK 21>`;
  run the tests with `./gradlew test`; build with `./gradlew build`; start the app from the root
  folder with `java -cp build/classes/java/main Main.Main` (it reads and writes `objectFile.txt` in
  the current folder). `gradle run` and `gradle test` are removed. The rest of the readme is unchanged.

**Evidence file:** add `## Slice 3` (the commands below, verbatim) and the final `## Not verified here`.

**Verification (builder records, verifier re-runs):**

| Command | Expected observed outcome |
|---|---|
| `grep -nE 'Gradle 6\|6\.7\|JDK 15\|≤ 15\|15 or older\|FizzBuzz\|gradle run\|gradle test\|committed, see' AGENTS.md readme.md` | no match |
| `grep -n '9.7.1\|JDK 21\|Main.Main' AGENTS.md readme.md` | each hit is backed by a line of the evidence file (verifier checks one by one) |
| `sed -n '8,12p;18,26p;30p' build.gradle` | the cited line ranges contain the toolchain, dependencies + test task, `Main-Class` |
| `/Users/ituser/Documents/tdli-ws/dev-agent-kit/bin/check .` | the four heading checks and the import check `ok`; the `../dev-agent-kit/WORKSPACE.md resolves` line fails only because the worktree is nested inside the repository (`dev-agent-kit/bin/check:32` resolves relative to the checked directory); the Lead re-runs `bin/check` on the merged commit (Ship) |
| `git diff --name-only HEAD~1 HEAD` | `AGENTS.md`, `readme.md`, the evidence file, optionally `docs/todo-list/README.md` |
| `git diff $BASE..HEAD -- src/ objectFile.txt dataFile.txt settings.gradle codegraph.toml scip-java.json .idea/` | empty |
| mechanical gate `JAVA_HOME=$J21 ./gradlew test --warning-mode all` | green, 14/0/0/0 (docs-only slice must not change it) |

**The verifier tries to break it:** a claim with no evidence line behind it (a JDK range, "works on
27" without an observed green, "the jar starts the app"); a line number in `AGENTS.md` that does not
point at what it names; a heading renamed or emptied (`bin/check`); the BUILD-1 text promising
auto-download or a JDK other than 21; the readme still telling users `gradle run`; `.idea/` or
`codegraph.toml` edited "for consistency".

## Evidence file

Path: **`docs/todo-list/evidence/2026-10-16-gradle-upgrade.md`** (dated the day slice 1 creates it,
per `docs/documentation-convention.md:24`; if slice 1 runs on a later day, that day's date). Listed
in `docs/todo-list/README.md § evidence` by slice 1 as
"`evidence/<date>-gradle-upgrade.md` — observed outputs of the gradle-upgrade slices (wrapper 9.7.1, gate on JDK 21, JDK 27 attempt)."

Format follows `docs/todo-list/evidence/2026-10-16-codegraph-onboarding.md:1-73` (contract:95-97):

```
# <date> — gradle-upgrade, observed build outputs

Observed outputs of named commands, run from the worktree `.claude/worktrees/gradle-upgrade`
on branch `feature/gradle-upgrade`, base `main` @ 46cd985. JAVA_HOME = <path> (<java -version first line>)
unless a command says otherwise.

## Slice 1 — <title>
### `<exact command>`
```<verbatim output, trimmed only at the ends; elisions marked "[… N lines elided …]">```
<one or two sentences of observation, no numbers that are not in the block above>

## Slice 2 — <title>
...
## Slice 3 — <title>
...

## Not verified here
```

Rules: every number is the output of a named command (convention rule 4); each slice appends only
its own section; failed commands (the baseline red, an offline resolution failure, a red JDK 27
attempt) are recorded verbatim like green ones. Long outputs (`--warning-mode all`, `dependencies`)
may be elided in the middle but keep the first and last 10 lines and every line containing
`deprecat`, `warning`, `FAILED`, `BUILD`, or a version number.

`## Not verified here` must at least cover: starting the application (agents do not run it, per the kit's
knowledge rule; contract:105-110 on SER-1 reading an old `objectFile.txt`);
`java -jar` on the built jar (manifest checked only); the JDK 27 result if it is not green; Gradle on
any JDK other than 21 and 27; the wrapper distribution checksum (not pinned); `.idea/` still
targeting JDK 15 (left alone by decision 4).

## Decisions left to implementation

None.

Everything the contract left open is settled above or by the human: target version, toolchain and
auto-download, `Main-Class` and no `run`, untracking and the `.gitignore` patterns, JUnit
coordinates, codegraph deferral (Settled decisions); `--enable-preview` and the duplicate
`useJUnitPlatform()` (deleted), `distributionSha256Sum` (not added), the slice order, the JDK 27
command, the offline-first resolution rule, the evidence path and format, and every docs sentence
(§ Interfaces, § Slices). A builder who finds a settled decision infeasible (see the risks marked
**stop**) returns `blocked`; that is a red gate, not a choice the builder makes.

## Risks

Each contract **[verify]** item is assigned to the slice that confirms it.

| # | Risk | Confirmed by | Mitigation / how it shows |
|---|---|---|---|
| R1 | The 6.7 launcher jar cannot start the 9.7.1 distribution, so the bootstrap fails (contract:188-190 [verify]). | slice 2 step 3 | The stash shows it worked on this machine (contract:85-94, :208-213). If it fails, the builder records the error and returns `blocked`; fallback (unzip the cached distribution and run its `bin/gradle wrapper`) is not pre-approved because it bypasses the wrapper. **stop** |
| R2 | A `distributionUrl` spelled differently from the cached one triggers a 9.7.1 download. | slice 2 | Exact spelling fixed in § Interfaces; verifier checks `~/.gradle/wrapper/dists/gradle-9.7.1-bin/` still has one directory. |
| R3 | `junit-platform-launcher` is not cached, so resolution needs Maven Central (architect's read of the cache). | slice 2 step 6 | Offline first, then online; the evidence states whether a download happened and what. No network = `blocked`. |
| R4 | The versionless engine and launcher do not align through the 5.6.0 BOM on Gradle 9 (contract:144-149, :173-174 [verify]). | slice 2 (`dependencies`) | Shows as an unresolved/`FAILED` entry. Pinning versions would be a change to settled decision 5: **stop**. |
| R5 | Gradle 9.7.1 rejects or misbehaves with JUnit Platform 1.6.0 (old launcher API), so the test task fails or finds 0 tests although resolution succeeds. Unknown; nothing on this machine shows a green 9.x test run (contract:196-199). | slice 2 (gate) | Shows as a red gate or fewer than 14 tests. A JUnit bump is excluded by settled decision 5: **stop**, with the verbatim error. |
| R6 | The toolchain is not satisfied by the running JDK 21, or Gradle tries to provision one (contract:272-273 [verify]). | slice 2 (`javaToolchains`, gate, `~/.gradle/jdks`) | `auto-download=false` plus no resolver plugin make provisioning impossible; failure shows as "No matching toolchains found", never as a download. **stop** if it occurs on JDK 21. |
| R7 | `org.gradle.java.installations.auto-download` is deprecated or renamed in 9.7.1, so `--warning-mode all` reports it, or it is silently ignored. | slice 2 (`--warning-mode all`, `javaToolchains`) | A deprecation line breaks the acceptance criterion: builder returns `blocked` with the suggested replacement from Gradle's message, since `gradle.properties` content is fixed here. **stop** |
| R8 | Other 9.7.1 deprecations in the new script (e.g. `test {}` / `jar {}` configuration style, `attributes` map form). | slice 2 | `--warning-mode all` shows them with the line; none is expected for the constructs kept (contract:167-180). Any line → `blocked` with Gradle's suggested form. **stop** |
| R9 | Gradle 9.7.1 does not run on JDK 27 (contract:200-206 [verify]). | slice 2 (one attempt) | Accepted outcome; recorded verbatim, and BUILD-1 / readme state it as observed. Not a stop. |
| R10 | `-Porg.gradle.java.installations.paths` is not accepted as a project property for toolchain discovery [verify]. | slice 2 (JDK 27 attempt) | The error is the recorded result of the single attempt; no second attempt (acceptance 4 says once). Not a stop. |
| R11 | `settings.gradle` needs a change on 9.x (contract:30 [verify]). | slice 2 (gate) | Not expected (`rootProject.name` only). If 9.7.1 demands a change, the file is frozen: **stop**. |
| R12 | `jcenter`, `compile`/`testCompile` removal and top-level `sourceCompatibility` deprecation (contract:171-172, :178-179 [verify]). | slice 2 (`build` exit 0, no deprecations) | Not used in the new script; `./gradlew build` exit 0 proves `compile` is gone. |
| R13 | A gate run modifies a tracked file (`objectFile.txt`, IO-1) or untracked leftovers get committed (`.gradle/9.7.1/`, `build/reports/problems/`, `bin/` from the experiment, contract:215-218). | slices 1-3 | Slice 1 makes them ignored; `git status --porcelain` must be empty after every gate run; the primary checkout's leftovers never reach the worktree. |
| R14 | A builder edits `src/` to quiet a javac 21 warning, breaking SER-1 (contract:105-110). | every slice | `git diff $BASE..HEAD -- src/` empty in every verify. |
| R15 | The worktree lacks the contract and design (they are uncommitted in the primary checkout). | before slice 1 | Precondition in § Shape: the Lead commits them; the slice-1 builder checks and returns `blocked` otherwise. |
| R16 | `bin/check` cannot fully pass inside the nested worktree. | slice 3 | Expected single failing line explained in slice 3; full `bin/check` on the merged commit at Ship. |
| R17 | Reading an `objectFile.txt` written by the JDK 15 build from a JDK 21 build (SER-1) is untested. | none (agents do not run the app) | Listed under "Not verified here"; `Task` is unchanged, so the serialized form is unchanged. |
