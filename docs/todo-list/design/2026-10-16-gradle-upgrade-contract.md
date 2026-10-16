# Change contract — gradle-upgrade

> Feature `gradle-upgrade`, repository `todo-list-java`, base `main` @ `46cd9856e9d589a2ce56ad2006bb20b7599372da`.
> Analyst: `analyst-gradle-upgrade`, roster `strict`, 2026-10-16.
>
> **Sources of facts.** Code facts come from the codegraph index via the kit CLI
> (`../dev-agent-kit/bin/codegraph tool ...`; the MCP server was not loaded in this session).
> `get_codegraph_summary` reports `todo-list-java @ 46cd9856e9d5 indexed 2026-10-16T13:03:19+00:00:
> modules=3 files=9 types=9 functions=57 calls=218`, the same revision as HEAD. The index covers Java
> sources only and excludes `.gradle/**`, `.idea/**` and `gradle/**` (`codegraph.toml:15`), so every
> fact about `build.gradle`, `settings.gradle`, the wrapper, `.gitignore`, the docs and committed
> build output comes from **repository reads** and read-only `git` commands, named where used.
> Nothing here was obtained by running `./gradlew` or the application.
>
> **Gradle claims.** Statements about what a Gradle version removes, requires or supports are not
> derivable from this repository. Each is marked **[verify]** and must be confirmed by the builder
> with the real wrapper before it is relied on. The one Gradle 9.7.1 behaviour observed on this
> machine is quoted from an artefact the earlier local experiment left behind (see § What changes,
> "Evidence from the stashed experiment").

## Affected subsystems

From `AGENTS.md § Subsystems` (`AGENTS.md:16-30`).

**Changes** (all three Gradle paths are frozen during fan-out per `AGENTS.md:51-54`; the Lead
unfreezes them explicitly for this feature):

| Subsystem row | Files | Why |
|---|---|---|
| `build.gradle`, `settings.gradle`, `gradle/wrapper/` (`AGENTS.md:26`) | `build.gradle`, `gradle/wrapper/gradle-wrapper.properties`, `gradle/wrapper/gradle-wrapper.jar`, `gradlew`, `gradlew.bat` | the upgrade itself. `gradlew` and `gradlew.bat` sit at the root but are wrapper output; `git ls-files` lists them alongside `gradle/wrapper/gradle-wrapper.jar` and `gradle/wrapper/gradle-wrapper.properties`. `settings.gradle` (one line, `settings.gradle:1`) needs no change for any target considered here **[verify]**. |
| `docs/` (`AGENTS.md:30`) | this contract, `docs/todo-list/README.md`, later the design and verify reports | documentation by feature |
| (not a subsystem row; agent guide and readme) | `AGENTS.md`, `readme.md` | BUILD-1, Mechanical gate, Build/run/test and the readme state Gradle 6.7 / JDK 15 (see § What changes, "Docs that change in the same feature") |

**Possibly changes, depending on open questions:**

| Subsystem row | Files | Trigger |
|---|---|---|
| `codegraph.toml`, `scip-java.json` (`AGENTS.md:29`; frozen, `AGENTS.md:53-54`) | `codegraph.toml:17-23`, `scip-java.json:1` | `codegraph.toml:19-21` says: "If the wrapper is ever upgraded, switch to build_tool = "gradle" and delete scip-java.json." Open question 6. |
| (not a subsystem row) committed build state | `build/` (25 tracked files), `.gradle/` (30 tracked files), `.gitignore` | open question 4 |

**Only read, must stay byte-identical:**

- `src/main/java/Main/` (`AGENTS.md:20`): `Main.java`, `UserInterface.java`.
- `src/main/java/TodoList/Task.java`, `TodoList.java`, `CompareTasks.java`, `FileReader.java` (`AGENTS.md:21-24`).
- `src/test/java/TodoList/` (`AGENTS.md:25`): `CompareTasksTest.java`, `TaskTest.java`, `TodoListTest.java`. The gate must go green **without editing a test**.
- `objectFile.txt`, `dataFile.txt`, `src/main/diagram.uml` (`AGENTS.md:27-28`).

### The test suite as it exists

Three classes, 14 `@Test` methods. Counts from `codegraph tool search_symbols query=Test` (14 `@Test`
rows, 3 `@BeforeEach`) and repository reads:

| Class | `@Test` methods (line of annotation) | JUnit imports |
|---|---|---|
| `CompareTasksTest` (`src/test/java/TodoList/CompareTasksTest.java:11`) | 1: `:20` | `org.junit.jupiter.api.BeforeEach`, `DisplayName`, `Test` (`:3-5`); static `Assertions.assertEquals` (`:9`) |
| `TaskTest` (`src/test/java/TodoList/TaskTest.java:11`) | 7: `:20`, `:31`, `:42`, `:53`, `:64`, `:75`, `:84` | `BeforeEach`, `DisplayName`, `Test` (`:3-5`); static `Assertions.assertEquals` (`:9`) |
| `TodoListTest` (`src/test/java/TodoList/TodoListTest.java:15`) | 6: `:24`, `:35`, `:44`, `:58`, `:73`, `:86` | `BeforeEach`, `DisplayName`, `Test` (`:3-5`); static `Assertions.assertEquals`, `assertTrue` (`:12-13`) |

- Every import is from `org.junit.jupiter.api` (the `junit-jupiter-api` artifact,
  `build.gradle:13`). No `@ParameterizedTest`, no `junit-jupiter-params`, no JUnit 4, no vintage
  engine, no mocking library (repository read of the three files).
- `TodoListTest.testRemoveTask` (`TodoListTest.java:35-42`) has an empty body; it passes trivially.
- No test touches the file system: `TodoListTest.setUp` uses the list constructor
  (`TodoListTest.java:21` → `TodoList.java:26-28`), not the default constructor that reads
  `objectFile.txt` through `FileReader` (`TodoList.java:20-23`). `find_callees` on
  `TodoList.TodoList.showTasksByProject`, `showTasksByDate`, `markAsDone`, `convertToDate` (the
  methods the tests call) returns only `Task` getters/setters, `CompareTasks` and JDK calls.
- **Baseline:** the committed reports from the last Gradle 6.x run show
  `CompareTasksTest tests="1"`, `TaskTest tests="7"`, `TodoListTest tests="6"`, all
  `failures="0" errors="0" skipped="0"`, timestamp `2021-03-19T17:53:14`
  (`build/test-results/test/TEST-TodoList.*.xml:2`, read with `head`). The upgraded gate must report
  **14 tests, 0 failures, 0 errors, 0 skipped**, observed fresh, not copied from these files.

## The existing pattern this change follows

There is no earlier build upgrade in this repository: `git log -- build.gradle
gradle/wrapper/gradle-wrapper.properties` shows two commits, `7e0347c` (2021-03-04, Initial commit)
and `1a47257` (2021-03-19). The change therefore follows three existing shapes:

1. **The build script's own idioms, which already match modern Gradle.** The `plugins { id 'java' }`
   block (`build.gradle:1-3`), `mavenCentral()` (`build.gradle:9`) and the
   `testImplementation` / `testRuntimeOnly` configuration names (`build.gradle:13-14`) are the
   current forms. Keep them; change only the constructs listed in § What changes. Do not rewrite
   the script to the Kotlin DSL or add plugins beyond what an open question approves.
2. **Wrapper files are generated, never hand-edited (except the bootstrap line).** The shape of what
   Gradle's `wrapper` task writes is evidenced by `stash@{0}` (read with `git diff stash@{0}^1
   stash@{0} -- <path>`; not applied): `gradle-wrapper.properties` gains `networkTimeout=10000`,
   `retries=0`, `retryBackOffMs=500`, `validateDistributionUrl=true` next to the new
   `distributionUrl`; `gradlew` (+201/−145 lines together with `gradlew.bat`) switches from
   `-classpath "$CLASSPATH" org.gradle.wrapper.GradleWrapperMain` (current `gradlew:83`, `:183`) to
   `-jar "$APP_HOME/gradle/wrapper/gradle-wrapper.jar"` (stash `gradlew:213`); and
   `gradle-wrapper.jar` changes from 59203 to 47505 bytes (`git stash show --stat stash@{0}`). The
   four wrapper files change together in one commit, produced by the `wrapper` task of the target
   version, not by copying from the stash.
3. **Evidence recording.** Observed gate output goes into an evidence file in the format of
   `docs/todo-list/evidence/2026-10-16-codegraph-onboarding.md:1-73`: the exact command, its
   verbatim output block, and a "Not verified here" section (`:68-73`).

## Invariants to preserve

From `AGENTS.md § Invariants` (`AGENTS.md:32-38`).

**Must hold untouched** (no file in their scope may change):

- `SER-1` (`AGENTS.md:34`, `Task.java:9`): `Task` keeps `serialVersionUID = 6076677258728022863L`
  and its fields. Risk from this change: none from the build script itself, since it adds no
  compiler flag that alters serialized form; the risk is a builder "tidying" `Task.java` while
  fixing warnings on a newer javac. Check: `git diff <base>..HEAD -- src/` is empty. Reading an
  existing `objectFile.txt` written by the JDK 15 build from a JDK 21 build is expected to work
  (same UID, same fields) but is **not verified here**, since agents do not run the application.
- `IO-1` (`AGENTS.md:35`, `FileReader.java:11`, `:30`): persistence stays relative to the working
  directory. Risk: only if the `application` plugin is added (open question 3); its `run` task's
  working directory must be the repository root **[verify]**. The gate itself never touches
  `objectFile.txt` (see the test-suite section above); `git status` after the gate must show
  `objectFile.txt` unchanged.
- `UI-1` (`AGENTS.md:36`, `UserInterface.java:84`, `TodoList.java:88`): unchanged code. Risk: only
  if a Gradle `run` task is added. Without forwarding stdin to the process, the menu's `Scanner`
  would see no input **[verify]**; the fix would be build configuration, never a source change.
- `DATE-1` (`AGENTS.md:37`, `TodoList.java:16`): unchanged code; covered by
  `TodoListTest.testConvertToDate` (`TodoListTest.java:86-94`), which must stay green.

**Changed by this feature** (rewritten in the same feature, by the slice that makes them true):

- `BUILD-1` (`AGENTS.md:38`): today "The wrapper pins Gradle 6.7, which runs only on JDK 15 or older".
  The rewrite states the new pinned version, the JDK range it runs on as observed by the builder,
  and (if open question 2 says yes) the toolchain version. It must not claim a JDK the builder did
  not observe.
- **Mechanical gate** (`AGENTS.md:40-47`): line 43 cites `build.gradle:16-22` and "Needs JAVA_HOME =
  a JDK ≤ 15 (BUILD-1)". Both become wrong. The command `./gradlew test` stays the gate; the
  comment changes to the new line range and the JDK requirement. Line 46 ("the readme's `gradle
  test` is the only documented check") changes if the readme is updated.

## Data and interfaces touched

- **No runtime data.** `objectFile.txt`'s format is unchanged (SER-1). No new dependency enters the
  main runtime classpath: `build.gradle:12-15` declares test dependencies only.
- **Build configuration:** `build.gradle`, the four wrapper files, possibly `.gitignore` (open
  question 4) and `codegraph.toml` / `scip-java.json` (open question 6).
- **External contract: the wrapper distribution URL** (`gradle/wrapper/gradle-wrapper.properties:3`,
  today `gradle-6.7-bin.zip` from `services.gradle.org`). The gate downloads the distribution on
  first run unless it is cached. `ls ~/.gradle/wrapper/dists` on this machine shows
  `gradle-6.7-bin`, `gradle-8.14.3-bin` and `gradle-9.7.1-bin`, each with a `.zip.ok` marker, so
  both candidate targets in open question 1 are already cached here.
- **Test dependency coordinates** (`build.gradle:13-14`): `org.junit.jupiter:junit-jupiter-api:5.6.0`
  and `org.junit.jupiter:junit-jupiter-engine` **without a version**. The versionless engine
  resolved under Gradle 6.x (the 2021 reports above prove the tests ran); the likely mechanism is
  the JUnit BOM referenced from `junit-jupiter-api`'s published Gradle module metadata, which
  aligns the engine to 5.6.0 **[verify]**: the builder must check `./gradlew dependencies
  --configuration testRuntimeClasspath` on the new version.
- **Bytecode level:** no `sourceCompatibility`, `targetCompatibility` or `java { toolchain }` is
  declared (grep of `build.gradle` and `settings.gradle` finds none), so classes are compiled for
  the JDK that runs Gradle. Observed: committed `build/classes/java/main/Main/Main.class` is class
  file major 59 (Java 15); the stash's copy is major 65 (Java 21) (bytes 6-7 read from `git show
  HEAD:<path>` and `git show stash@{0}:<path>`). Any consumer of the classes needs a JDK ≥ the
  compile JDK; `AGENTS.md:62` runs them with `java -cp build/classes/java/main Main.Main`.
- **Codegraph indexing:** unaffected unless open question 6 switches it; the index uses
  `build_tool = "scip"` with javac from `JAVA_HOME` (`codegraph.toml:17-23`).

## What changes

### Construct-by-construct review of `build.gradle` and `settings.gradle`

Read in full: `build.gradle` (34 lines), `settings.gradle` (1 line). A grep for `compile`,
`testCompile`, `testRuntime` (other than `testRuntimeOnly`), `jcenter`, `sourceCompatibility`,
`targetCompatibility`, `toolchain`, `application`, `mainClass` matches only `build.gradle:32`.

| Location | Construct | Status under Gradle 7 / 8 / 9 | Action |
|---|---|---|---|
| `build.gradle:1-3` | `plugins { id 'java' }` | current form | keep |
| `build.gradle:5`, `:6` | `group 'org.example'`, `version '1.0-SNAPSHOT'` (space-assignment) | **Observed** on Gradle 9.7.1: two deprecation problems at `build.gradle` lines 5 and 6, "Properties should be assigned using the 'propName = value' syntax ... has been deprecated", details "This is scheduled to be removed in Gradle 10", solutions "Use assignment ('group = <value>') instead." / "('version = <value>')" (untracked `build/reports/problems/problems-report.html`, extracted with a Python regex over the embedded JSON). Whether 8.x warns depends on the 8.x minor **[verify]**. | change to `group = 'org.example'`, `version = '1.0-SNAPSHOT'` so the gate runs without deprecations |
| `build.gradle:8-10` | `repositories { mavenCentral() }` | current. `jcenter()` is **absent**, so its deprecation in 7.x and removal later **[verify]** do not apply | keep |
| `build.gradle:13` | `testImplementation 'org.junit.jupiter:junit-jupiter-api:5.6.0'` | configuration name current; `compile` / `testCompile` / `testRuntime` (removed in 7.0 **[verify]**) are **not** used in `dependencies {}` | keep the name; version per open question 5 |
| `build.gradle:14` | `testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine'` (no version) | resolves through the BOM alignment described above **[verify]** | keep, or replace both lines with a `platform('org.junit:junit-bom:<v>')` + versionless artifacts (open question 5) |
| (absent) | `testRuntimeOnly 'org.junit.platform:junit-platform-launcher'` | Gradle 8.x deprecated relying on Gradle to supply the JUnit Platform launcher automatically, and Gradle 9 requires it on the test runtime classpath; without it the test task fails to load the JUnit Platform **[verify]**. This is the most likely single cause of a red gate on 9.x. | add, versionless so the JUnit BOM aligns it (1.6.0 for Jupiter 5.6.0) **[verify]** |
| `build.gradle:17-19` and `:21-24` | `useJUnitPlatform()` called twice (in `test {}` and in `tasks.withType(Test) {}`) | both calls are the same framework, so redundant rather than broken; whether 8.x/9.x warn on re-selecting the framework **[verify]** | keep exactly one `useJUnitPlatform()`; the gate depends on it being present |
| `build.gradle:23` | `jvmArgs '--enable-preview'` on test JVMs | not a Gradle issue. Nothing compiles with `--enable-preview` and a grep of `src/` for `record `, `sealed `, text blocks, arrow-switch finds no preview-era syntax, so the flag is unused. A JVM accepts `--enable-preview` for classes compiled without preview **[verify on 21 and 27]** | remove (dead config), or keep if the architect prefers a zero-risk diff. Removing it does not change test behaviour |
| `build.gradle:26-29` | `jar { manifest { attributes "Main-Class": "com.gradle.tutorial.FizzBuzzProcessor" } }` | valid Gradle; wrong value. `codegraph tool search_symbols query=FizzBuzz` → 0 results; the real entry point is `Main.Main.main` (`src/main/java/Main/Main.java:4`). Already documented at `AGENTS.md:65-67`. | open question 3 |
| `build.gradle:31-33` | `jar { from { configurations.compile.collect { ... zipTree(it) } } }` | the `compile` configuration was removed in Gradle 7.0 **[verify]**; this is the only use of it. The closure is lazy, so `./gradlew test` likely never evaluates it (the `jar` task is not a dependency of `test`), while `./gradlew build` / `assemble` / `jar` would fail with an unknown property `compile` **[verify]**. With no main dependencies the fat-jar block adds nothing today. | replace with `configurations.runtimeClasspath` or delete the block (it is a no-op); must not be left referencing `compile`, so that `./gradlew build` (`AGENTS.md:60`) also works |
| (absent) | `sourceCompatibility` / `java { toolchain { ... } }` | top-level `sourceCompatibility` is not used, so its deprecation in favour of `java { }` does not apply **[verify]** | open question 2 |
| `settings.gradle:1` | `rootProject.name = 'todolist'` | current | keep |

### Wrapper

- `gradle/wrapper/gradle-wrapper.properties:3` pins `gradle-6.7-bin.zip`. Gradle 6.7 cannot run on
  JDK 21: observed in `docs/todo-list/evidence/2026-10-16-codegraph-onboarding.md:70-72`
  ("Unsupported class file major version 65"), and no JDK ≤ 15 exists on this machine (Lead's
  machine facts). The upgrade is therefore a bootstrap: the 6.7 wrapper cannot run its own
  `wrapper` task here. The usual route is to set `distributionUrl` to the target version first,
  then run `./gradlew wrapper --gradle-version <target>` on JDK 21 so the target version
  regenerates `gradlew`, `gradlew.bat` and `gradle-wrapper.jar` **[verify]**. The builder records
  the exact commands in the evidence file. Adding `distributionSha256Sum` is optional; architect's
  call.
- **JDK 21:** the Gradle versions considered (latest 9.x, or 8.14.x) are documented by Gradle as
  running on Java 21, and Gradle 9 requires Java 17+ to run **[verify]**. Partial on-machine
  evidence: the stashed 9.7.1 experiment ran on this machine far enough to write
  `build/classes/java/main` as class file major 65 and to emit the problems report. It did **not**
  rewrite the test classes or test reports (stash file list has no `build/classes/java/test` or
  `build/test-results` entries; `build/test-results/test/*.xml` still carry the 2021 timestamp), so
  **no green `./gradlew test` on any upgraded version has been observed yet**.
- **JDK 27:** realistic only if the chosen Gradle version supports *running* on Java 27, or if
  Gradle runs on 21 and compiles/tests through a toolchain set to 27. Neither can be established
  from this repository. Java 27 is newer than any Gradle 8.x, so 8.x on 27 is unlikely; 9.7.1 on
  27 is unknown **[verify]**. The contract's acceptance criterion is JDK 21; JDK 27 is
  "attempted and recorded" (observed result in the evidence file), not required. `/opt/homebrew/opt/openjdk`
  is a symlink to `../Cellar/openjdk/27` (`ls -la`), so a bare Homebrew `openjdk` on `PATH` means
  27, not 21; the gate command in the docs must say which `JAVA_HOME` it was observed with.

### Evidence from the stashed experiment (read, not applied)

`git stash show --stat stash@{0}` lists 15 files: the four wrapper files, six
`build/classes/java/main/**.class` files and five `.gradle/` lock/cache files, including
`.gradle/buildOutputCleanup/cache.properties` changed from `gradle.version=6.7` to
`gradle.version=9.7.1`. So a wrapper run rewrites **tracked** files under `build/` and `.gradle/`.
That is why open question 4 matters: every gate run on the upgraded build will produce diffs in
tracked build state unless it is untracked. The working tree also holds untracked leftovers of the
same experiment (`git status --short`: `.gradle/9.7.1/`, `build/reports/problems/`, and `bin/`,
an IDE output folder with `main/` and `test/`); they are not part of this feature and the builder
must not commit them.

### Committed build output and `.gradle/` (scope question)

- `git ls-files build | wc -l` → 25; `git ls-files .gradle | wc -l` → 30 (under `.gradle/6.7/`,
  `.gradle/6.8.3/`, `.gradle/buildOutputCleanup/`, `.gradle/checksums/`, `.gradle/configuration-cache/`,
  `.gradle/vcs-1/`); `git ls-files .idea | wc -l` → 10.
- `.gitignore:1-3` contains three lines, each a `.DS_Store` variant with a pasted shell command
  appended (`.DS_STORE $ git rm --cached .DS_Store rm .DS_Store $ git commit -m Start ignoring
  .DS_store`). `git check-ignore -v build/x .gradle/x bin/x .DS_Store` reports nothing ignored. So
  `build/` and `.gradle/` are tracked because nothing ignores them. `AGENTS.md:60` ("committed, see
  .gitignore") describes this state.
- `.idea/misc.xml:14` (`languageLevel="JDK_15"`, `project-jdk-name="15"`) and
  `.idea/compiler.xml:4` (`bytecodeTargetLevel target="15"`) pin the IDE to JDK 15. They are not in
  the gate's path; this contract leaves them out of scope unless the human folds them into open
  question 4.

### Docs that change in the same feature

| File | Lines | Change |
|---|---|---|
| `AGENTS.md` | `:13-14` | "Gradle 6.7 wrapper build" → new version |
| `AGENTS.md` | `:26` | subsystem row "Gradle 6.7 build of root project `todolist`" → new version |
| `AGENTS.md` | `:38` | `BUILD-1` rewritten (see § Invariants) |
| `AGENTS.md` | `:43`, `:46` | Mechanical gate comment: new `build.gradle` line range, JDK requirement |
| `AGENTS.md` | `:58-63` | Build/run/test: `export JAVA_HOME=<JDK 15 or older>` → the observed JDK 21 path form; `build/` "committed" note per open question 4 |
| `AGENTS.md` | `:65-67` | only if open question 3 fixes `Main-Class` / adds `run` |
| `readme.md` | `:6-10` | "Build with ... JDK 15" → JDK 21 |
| `readme.md` | `:20-32` | `gradle run` / `gradle test` → `./gradlew test`; `run` only if open question 3 adds it |
| `docs/todo-list/README.md` | `:6-8` design list, `## evidence` | list this contract (done in this change), the design, and the builder's evidence / verify reports |
| `codegraph.toml` | `:17-23` comments | only if open question 6 is "now"; otherwise the comment at `:19-21` stays accurate as a pending note |

### Acceptance (what the verifier checks)

1. `JAVA_HOME=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home ./gradlew test` exits 0
   and reports 14 tests, 0 failures, 0 errors, 0 skipped (from the fresh
   `build/test-results/test/*.xml`), with no deprecation warnings from `build.gradle` (`--warning-mode all`).
2. `./gradlew build` on the same JDK also exits 0 (proves `build.gradle:31-33` no longer references
   `compile`).
3. `git diff <base>..HEAD -- src/ objectFile.txt dataFile.txt` is empty (SER-1, IO-1, UI-1, DATE-1).
4. The same gate on JDK 27 is attempted once and its observed result recorded, green or not.
5. `AGENTS.md` BUILD-1, Mechanical gate, Build/run/test and `readme.md` match what was observed.

## Open questions

1. **Target Gradle version.** Latest 9.x (9.7.1 is cached here and was tried locally; it already
   flags `build.gradle:5-6` as removed in Gradle 10) or a conservative 8.x (8.14.3 is also cached).
   9.x needs the explicit JUnit Platform launcher **[verify]** and gives a longer runway; 8.14.x is
   likely more forgiving with this 2021 script but is the end of a line, and running it on JDK 27
   is unlikely **[verify]**. Analyst's lean: latest 9.x, since the script changes required are small
   and the same either way.
2. **Toolchain.** Add `java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }`, pinning
   compile and test to Java 21 regardless of which JDK runs Gradle (and making a "Gradle on 27" run
   still test on 21), or leave the bytecode level implicit (= the JDK running Gradle, today 21)?
   A toolchain also lets Gradle auto-provision a JDK when 21 is absent, which touches the network
   **[verify]**. Related: is "JDK 27 attempted and recorded" enough, or must 27 be green?
3. **Jar `Main-Class` and `application` plugin.** Fix `build.gradle:28` to `Main.Main` and/or add
   the `application` plugin with `mainClass = 'Main.Main'` (so `./gradlew run` matches the readme)
   in this feature, or defer to a separate feature? Adding `run` brings IO-1 (working directory)
   and UI-1 (stdin forwarding) into scope **[verify]**. Independently of the answer, the
   `configurations.compile` reference at `build.gradle:32` must go so `./gradlew build` works.
4. **Untrack `build/` and `.gradle/`.** Remove the 25 + 30 tracked files with `git rm --cached` and
   replace the malformed `.gitignore:1-3` with real patterns (`build/`, `.gradle/`, `.DS_Store`,
   possibly `bin/` and `out/`), in this feature or a separate one? If deferred, every gate run
   produces diffs in tracked files (shown by the stash) that the builder must not commit. Also: do
   `.idea/misc.xml:14` / `.idea/compiler.xml:4` (JDK 15) get updated, untracked, or left alone?
5. **JUnit version.** Keep `junit-jupiter-api:5.6.0` (`build.gradle:13`) and add only the launcher,
   or move to a `junit-bom` platform at a current 5.x version? The tests use only basic Jupiter API
   (§ Affected subsystems, test suite), so a bump is low risk, but it is a change the gate does not
   strictly need.
6. **Codegraph indexing route.** `codegraph.toml:19-21` asks to switch to `build_tool = "gradle"` and
   delete `scip-java.json` once the wrapper is upgraded (both frozen, `AGENTS.md:53-54`). Do that in
   this feature (requires a re-index and an onboarding-style evidence update), or defer to a
   follow-up and leave the scip route, which works today?
