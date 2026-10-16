@../dev-agent-kit/WORKSPACE.md

# todo-list-java — agent guide

Compact map for agents working in this repository. Do not repeat what the code shows; point at
the file. Rationale lives in `docs/<feature>/guides/`. The four headed sections below are read
by the dev-agent-kit orchestration skill and checked by `bin/check`; keep their exact headings.

## What this is

ToDoLy: a console to-do list in plain Java (no frameworks). A numbered menu loop reads choices
from stdin; tasks (title, due date, user, project, done flag) live in an `ArrayList<Task>` and
are persisted by Java serialization to `objectFile.txt` in the working directory. Gradle 9.7.1
wrapper build with a Java 21 toolchain, JUnit 5 tests, no CI. Usage walkthrough: `readme.md`.

## Subsystems

| Path | Responsibility |
|---|---|
| `src/main/java/Main/` | entry point `Main` and the menu loop `UserInterface` (one static `TodoList`) |
| `src/main/java/TodoList/Task.java` | the serializable task record with getters/setters |
| `src/main/java/TodoList/TodoList.java` | task list operations, console prompts, date parsing, `save()` |
| `src/main/java/TodoList/CompareTasks.java` | due-date comparator used by `showTasksByDate` |
| `src/main/java/TodoList/FileReader.java` | serialize/deserialize the list to/from `objectFile.txt` |
| `src/test/java/TodoList/` | JUnit 5 tests for `Task`, `TodoList`, `CompareTasks` |
| `build.gradle`, `settings.gradle`, `gradle.properties`, `gradle/wrapper/` | Gradle 9.7.1 build of root project `todolist`, Java 21 toolchain, no JDK auto-download |
| `objectFile.txt` | the serialized task store the app writes at runtime (committed) |
| `dataFile.txt`, `src/main/diagram.uml` | legacy text dump and the class-diagram source; no code reads them |
| `codegraph.toml`, `scip-java.json` | codegraph index config (kit); see `codegraph.toml` comments |
| `docs/` | documentation by feature area (`docs/documentation-convention.md`) |

## Invariants

- `SER-1` `Task` is `Serializable` with a fixed `serialVersionUID` (`Task.java:9`); changing its fields or the UID makes an existing `objectFile.txt` unreadable.
- `IO-1` All persistence goes through `FileReader`; the path `objectFile.txt` is relative to the working directory (`FileReader.java:11`, `:30`), so the app must run from the repository root to see saved tasks.
- `UI-1` Menu choices are `Integer.parseInt` on raw stdin lines (`UserInterface.java:84`, `TodoList.java:88`); nothing catches `NumberFormatException`, so non-numeric input ends the process.
- `DATE-1` Dates are entered and parsed as `dd/MM/yyyy` (`TodoList.java:16`); `convertToDate` returns null on failure and `validateDate` loops until it parses.
- `BUILD-1` The wrapper pins Gradle 9.7.1 (`gradle/wrapper/gradle-wrapper.properties:3`), `build.gradle:8-12` pins a Java 21 toolchain and `gradle.properties` disables JDK auto-download, so Gradle must find an installed JDK 21. The gate is observed green with JAVA_HOME = JDK 21; with JAVA_HOME = JDK 27 it is green only with `-Dorg.gradle.java.installations.paths=<JDK 21>`, otherwise it fails with "Cannot find a Java installation … matching: {languageVersion=21 …}".

## Mechanical gate

```
./gradlew test        # JUnit 5 via the wrapper (build.gradle:18-26). Needs JAVA_HOME = a JDK 21 (BUILD-1).
```

There is no CI workflow and no linter or formatter; `./gradlew test` (also in the readme) is the
only documented check. Changes under `src/main/java/TodoList/` must keep `src/test/java/TodoList/` green.

## Branching

Default branch `main`. Feature branches, PRs into `main`, merge commits (see `git log`). Frozen
during fan-out (Lead-owned; builders request changes by message): `build.gradle`,
`settings.gradle`, `gradle.properties`, `gradle/wrapper/`, `objectFile.txt`, `dataFile.txt`,
`codegraph.toml`, `scip-java.json`.

## Build, run, test

```
export JAVA_HOME=<a JDK 21>             # e.g. /opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home (BUILD-1)
./gradlew build                         # compiles into build/ (ignored, .gitignore:1)
./gradlew test
java -cp build/classes/java/main Main.Main   # run from the repository root (IO-1)
```

There is no `run` task: `build.gradle` applies only the `java` plugin, not `application`; start
the app with the `java -cp` line above. The jar manifest names `Main.Main` as `Main-Class`
(`build.gradle:30`); starting the app from the jar is not verified by agents.

## Maintaining this file

Keep it a map. Prefer pruning over appending. Move rationale to `docs/<feature>/guides/` and link it.
