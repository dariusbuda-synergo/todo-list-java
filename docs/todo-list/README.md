# todo-list

The ToDoLy command-line application: the only feature area in this repository. Layout per
`../documentation-convention.md`.

## design

- `design/2026-10-16-gradle-upgrade-contract.md` — change contract for feature `gradle-upgrade`: upgrade the Gradle 6.7 build so `./gradlew test` runs on JDK 21.
- `design/2026-10-16-gradle-upgrade-design.md` — design for feature `gradle-upgrade`: Gradle 9.7.1 with a Java 21 toolchain in three slices (untrack build state, upgrade to green on JDK 21, docs).

## evidence

- `evidence/2026-10-16-codegraph-onboarding.md` — observed counts from the first codegraph index of this repository.
- `evidence/2026-10-16-gradle-upgrade.md` — observed outputs of the gradle-upgrade slices (wrapper 9.7.1, gate on JDK 21, JDK 27 attempt).
- `evidence/gradle-upgrade-1.verify.md` — verify report for gradle-upgrade slice 1 (untrack build state, `.gitignore`): green; gate red on JDK 21 as the design predicts.
- `evidence/gradle-upgrade-2.verify.md` — verify report for gradle-upgrade slice 2 (Gradle 9.7.1 wrapper, build script, gate on JDK 21): green; 14/0/0/0 on JDK 21, JDK 27 green with the JDK 21 path, fails without it.

## guides

- `guides/class-diagram.png` — class diagram of the application (source: `src/main/diagram.uml`).
- Usage walkthrough: the repository `readme.md`.
