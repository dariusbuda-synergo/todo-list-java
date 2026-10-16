# 2026-10-16 — codegraph onboarding, first index

Observed outputs of named commands, run from the repository root on branch
`chore/agentic-kit-setup` (working tree at `main` revision `71f955b`), after
`../dev-agent-kit/adapters/claude-code/install.sh`. Tooling: scip-java 0.13.1 (`--build-tool scip`,
sources listed in `scip-java.json`), JAVA_HOME = Homebrew openjdk@21 (javac 21.0.12.1).

## `../dev-agent-kit/bin/check`

```
-- todo-list-java
  ok   imports ../dev-agent-kit/WORKSPACE.md
  ok   ../dev-agent-kit/WORKSPACE.md resolves
  ok   ## Subsystems (13 lines)
  ok   ## Invariants (5 lines)
  ok   ## Mechanical gate (5 lines)
  ok   ## Branching (4 lines)
  ok   documentation-convention.md identical
  ok   .mcp.json registers codegraph
  ok   skill: ~/.claude/skills/orchestration -> /Users/ituser/Documents/tdli-ws/dev-agent-kit/orchestration
RESULT: GREEN
```

## `../dev-agent-kit/bin/codegraph index`

```
loaded 9 documents, 1074 occurrences
mapped todo-list-java: {'files_excluded': 0, 'definitions_duplicate_dropped': 2, 'calls_definition': 81,
  'calls_external': 135, 'calls_module_level_dropped': 4, 'calls_unresolved': 2, 'inheritance_external': 2,
  'modules': 3, 'files': 9, 'types': 9, 'functions': 57, 'calls': 218, 'inheritance': 2}
projected codegraph:shadow -> codegraph: {'Repository': 1, 'Module': 3, 'File': 9, 'Type': 9,
  'Function': 57, 'Generation': 1, 'CONTAINS': 12, 'DECLARES': 123, 'CALLS': 76, 'INHERITS': 0}
exit 0
```

`git status` after the run shows only the onboarding files: scip-java writes nothing into the
repository (targetroot is under the kit's `.scip-cache/`). javac reports 67 errors while
compiling `src/test/java` because no JUnit classpath is given; scip-java still exits 0 and the
test files are indexed (9 documents = 6 main + 3 test).

## `../dev-agent-kit/bin/codegraph tool get_codegraph_summary`

```
Repositories: 1 result
- todo-list-java @ 71f955bab8d8 indexed 2026-10-16T12:58:52+00:00: modules=3 files=9 types=9
  functions=57 calls=218 (definition=81 external=135 unresolved=2; module-level dropped=4
  import-line dropped=0) inheritance=2
```

## `../dev-agent-kit/bin/codegraph tool find_callers identifier=TodoList.TodoList.save`

```
Callers of TodoList.TodoList.save: 4 results
- public void showTasksMenu() { (src/main/java/Main/UserInterface.java:70) resolved_by=definition
- public void run() { (src/main/java/Main/UserInterface.java:102) resolved_by=definition
- public void editTaskOptions(Task task) { (src/main/java/TodoList/TodoList.java:160) resolved_by=definition
- public void addTask() { (src/main/java/TodoList/TodoList.java:207) resolved_by=definition
```

## `../dev-agent-kit/bin/codegraph tool find_callees identifier=Main.UserInterface.run`

```
Callees of Main.UserInterface.run (resolved): 5 results
Callees of Main.UserInterface.run (external or unresolved): 8 results
(13 call sites total)
```

## Not verified here

`./gradlew test` (the mechanical gate) could not be run on this machine: the wrapper's Gradle 6.7
fails on JDK 21 with `Unsupported class file major version 65` (observed on a scratch copy of the
repository). It needs a JDK 15 or older. The main sources compile with javac 21 and the app
starts and exits cleanly on option 4 (observed on a scratch copy).
