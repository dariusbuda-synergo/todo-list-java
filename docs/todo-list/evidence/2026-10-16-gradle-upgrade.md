# 2026-10-16 — gradle-upgrade, observed build outputs

Observed outputs of named commands, run from the worktree `.claude/worktrees/gradle-upgrade`
on branch `feature/gradle-upgrade`, base `main` @ 46cd985. JAVA_HOME = $J21 =
`/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home` (`openjdk version "21.0.12.1" 2026-08-18`)
unless a command says otherwise. `BASE=46cd9856e9d589a2ce56ad2006bb20b7599372da`.

## Slice 1 — untrack build state and fix `.gitignore`

Changes made: `git rm -r --cached --quiet build .gradle` (index only), `.gitignore` rewritten to the
five lines of the design, this file created and listed in `docs/todo-list/README.md`.

The design's `HEAD~1 HEAD` checks were run before the commit, on the staged change, as
`git diff --cached --name-status HEAD` (index against the parent commit `d892696`); the verifier
re-runs them as `HEAD~1 HEAD` on the commit. `$BASE..HEAD` was run as `git diff --cached $BASE`
for the same reason.

### `$J21/bin/java -version 2>&1 | head -1`

```
openjdk version "21.0.12.1" 2026-08-18
```

The JDK used for every command below that runs Java.

### `JAVA_HOME=$J21 ./gradlew test; echo "exit=$?"`

```
Starting a Gradle Daemon, 3 incompatible Daemons could not be reused, use --status for details

FAILURE: Build failed with an exception.

* Where:
Settings file '/Users/ituser/Documents/tdli-ws/todo-list-java/.claude/worktrees/gradle-upgrade/settings.gradle'

* What went wrong:
Could not compile settings file '/Users/ituser/Documents/tdli-ws/todo-list-java/.claude/worktrees/gradle-upgrade/settings.gradle'.
> startup failed:
  General error during semantic analysis: Unsupported class file major version 65
  
  java.lang.IllegalArgumentException: Unsupported class file major version 65
[… 211 lines elided …]
  
  1 error

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 1s
exit=1
```

Baseline, recorded not fixed: the Gradle 6.7 wrapper fails at startup on JDK 21 with `Unsupported class file major version 65` while compiling `settings.gradle`, as on the base. Slice 1 does not touch the build; the gate is red as the design expects for this slice.

### `git ls-files build .gradle | wc -l`

```
       0
```

Nothing under `build/` or `.gradle/` is tracked any more.

### `git diff --cached --name-status HEAD | grep -vE '^D\s+(build|\.gradle)/'`

```
M	.gitignore
M	docs/todo-list/README.md
A	docs/todo-list/evidence/2026-10-16-gradle-upgrade.md
```

Apart from the index removals, the change touches exactly `.gitignore`, the README and this file.

### `git diff --cached --name-status HEAD | grep -cE '^D\s+(build|\.gradle)/'`

```
55
```

### `git diff --cached --name-status HEAD | grep -E '^D\s+build/' | wc -l`

```
      25
```

### `git diff --cached --name-status HEAD | grep -E '^D\s+\.gradle/' | wc -l`

```
      30
```

The removals split into the two directories as the contract counted them.

### `git check-ignore -v build/x .gradle/x bin/x out/x .DS_Store src/main/.DS_Store`

```
.gitignore:1:build/	build/x
.gitignore:2:.gradle/	.gradle/x
.gitignore:4:bin/	bin/x
.gitignore:5:out/	out/x
.gitignore:3:.DS_Store	.DS_Store
.gitignore:3:.DS_Store	src/main/.DS_Store
```

Each probe path is ignored by its own line of `.gitignore`, including a nested `.DS_Store`.

### `git check-ignore gradle/wrapper/gradle-wrapper.jar gradlew gradlew.bat gradle.properties .idea/misc.xml src/main/java/Main/Main.java objectFile.txt dataFile.txt; echo $?`

```
1
```

No path printed: the wrapper, `gradle.properties`, `.idea/`, sources and the data files are not ignored.

### `git ls-files -ci --exclude-standard`

```

```

Empty: no tracked file is matched by an ignore rule.

### `cat -vet .gitignore`

```
build/$
.gradle/$
.DS_Store$
bin/$
out/$
```

Five lines, LF endings, no leading `/`, no stray text.

### `test -d build && test -d .gradle && echo present`

```
present
```

Only the index changed; the directories stay on disk.

### `git diff --cached $BASE -- src/ objectFile.txt dataFile.txt settings.gradle build.gradle gradle/ gradlew gradlew.bat codegraph.toml scip-java.json .idea/`

```

```

Empty: none of the frozen or slice-2 files changed.

### `git ls-files -s gradlew`

```
100755 4f906e0c811fc9e230eb44819f509cd0627f2600 0	gradlew
```

### `git status --porcelain | grep -vE '^D  (build|\.gradle)/'`

```
M  .gitignore
M  docs/todo-list/README.md
A  docs/todo-list/evidence/2026-10-16-gradle-upgrade.md
```

Run after the gate, before the commit: only the staged slice-1 changes; the gate run left no untracked or modified file (its output lands in ignored directories).
