# AGENTS.md

Notes for automated agents working in this repository (Base44 sandbox).

## What this repository is

TheAlgorithms/Java is a **library**, not an application: plain algorithm
implementations in `src/main/java/com/thealgorithms/**` with JUnit tests in
`src/test/java`. There is no server, no entry point, no database and no
environment or secret configuration.

## Environment

- Start: `docker compose -f docker-compose.base44.yml up -d`
- Stop: `docker compose -f docker-compose.base44.yml down`
- The single service `preview` (`maven:3.9-eclipse-temurin-21`) keeps the repo
  bind-mounted at `/app` and on every start runs `mvn compile`, then compiles and
  runs `.base44/preview/PreviewServer.java.in` against `target/classes` on host
  port 3000.
- Maven artifacts are cached in the `maven-repo` volume, so the first start
  downloads the dependency/plugin set (several minutes) and later starts are fast.

## Things that are easy to get wrong

- **No hot reload.** The library is compiled when the container starts. After
  editing anything under `src/main/java`, run
  `docker compose -f docker-compose.base44.yml restart preview` (the preview then
  needs a reload).
- **Never add a `*.java` file outside `src/main/java` or `src/test/java`.**
  `.github/workflows/scripts/check_structure.py` fails CI on any misplaced Java
  file. That is why the preview harness is committed as
  `.base44/preview/PreviewServer.java.in` and copied to `/tmp` at start-up — keep
  that extension.
- The harness must only call **public** API. Many algorithm classes are
  package-private (`QuickSort`, `MergeSort`, `BinarySearch`, `Palindrome`) and are
  not reachable from outside their package.
- `pom.xml` compiles with `-Werror -Xlint:all`, so a warning is a build failure.

## Verifying changes

Everything runs inside the container — the sandbox host has no JDK.

```bash
# compile the library only
docker compose -f docker-compose.base44.yml exec -T preview mvn --batch-mode --quiet compile

# run a single test class (the full suite is very large)
docker compose -f docker-compose.base44.yml exec -T preview mvn --batch-mode -Dtest=HeapSortTest test

# exercise the preview harness
curl -s -X POST --data '5, 3, 9, 1' http://localhost:3000/api/run?algo=heapsort
curl -s http://localhost:3000/health
```

Quality gates used by CI (run them only for files you touch):
`mvn checkstyle:check`, `mvn spotbugs:check`, `mvn pmd:check`.

## Known quirks

- `PreviewServer.java.in` writes HTML/JSON by hand; user input is escaped in both
  paths. Keep that when adding demos.
