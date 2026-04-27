# AGENTS.md

Two co-located modules sharing one Git repo — no workspace manager ties them together. Each is built and run independently.

- `server/` — Spring Boot 3 / Java 25 REST API (Gradle Kotlin DSL)
- `client/` — SvelteKit 2 / Svelte 5 frontend (Vite + Bun)
- `scripts/` — Cross-module shell utilities

---

## Environment Setup

Use `mise` to install exact tool versions declared in `mise.toml` (Java 25 Temurin, Bun 1.3.0, Node 22.20.0):

```bash
mise install
```

`mise.toml` also exports `SPRING_PROFILES_ACTIVE=dev` and `SERVER_PORT=8080` automatically when using `mise`.

---

## Server (`server/`)

All commands run from the `server/` directory.

```bash
./gradlew bootRun          # Dev server (H2, Liquibase drop-first, user/password auth)
./gradlew test             # All tests
./gradlew test --tests "dev.ilionx.workshop.api.vet.controller.VetControllerTest"  # Single class
./gradlew test --tests "dev.ilionx.workshop.api.vet.controller.VetControllerTest.methodName"  # Single method
./gradlew check            # Checkstyle + PMD + SpotBugs + Codenarc
./gradlew spotlessApply    # Format code (also runs automatically before every compile)
./gradlew spotlessCheck    # Check formatting without writing (use this in CI)
./gradlew bootJar          # Build fat JAR
```

**Quirks:**
- `JavaCompile` depends on `spotlessApply` — every build auto-formats source files.
- `-Werror` on `javac` — all compiler warnings fail the build.
- Context path is `/api`, so REST endpoints are at `http://localhost:8080/api/v1/...`. Tests override context path to `""` via `application-test.yml`.
- Dev profile (`application-dev.yml`) sets `drop-first: true` — database is torn down and re-seeded with test data on every server boot.
- Liquibase contexts: `prd` for production, `tst` for tests and dev.
- All dependency versions are declared in `gradle.properties`, not inline in `build.gradle.kts`. Use the `retrieve()` helper there.

**MapStruct + Lombok:** Lombok must appear before `mapstruct-processor` in the annotation processor list (already correct — do not reorder).

### Domain Package Convention

Each domain (`owner`, `pet`, `vet`, `visit`) follows a strict layout:
```
api/<domain>/
  controller/
  service/
  repository/
  model/
    request/
    response/
    mapper/
```

- All URL path strings are constants in `dev.ilionx.workshop.api.Paths` — never hardcode paths.
- Unit tests extend `UnitTest` (Mockito only, no Spring context).
- Integration tests extend `IntegrationTest` (full Spring Boot context, `@ActiveProfiles("test")`, H2, random port, MockMvc). The base class cleans the database before/after each test but preserves seeded rows (IDs ≤ 6 for vets/pet types, IDs ≤ 3 for specialties). **Do not alter or remove TST Liquibase changeset rows for these IDs** — integration tests depend on them.

---

## Client (`client/`)

All commands run from the `client/` directory. Use `bun`, not `npm` or `yarn` (`bun.lock` is the authoritative lockfile).

```bash
bun run dev            # Dev server at http://localhost:5173
bun run build          # Production build
bun run check          # svelte-check type-check (only client verification step — no unit tests exist)
bun run check:watch    # Type-check in watch mode
bun run sync:api       # Boot server, download OpenAPI spec, regenerate TS types (full pipeline)
bun run generate:api   # Generate TS types from existing openapi.json only
```

**Quirks:**
- `src/lib/types/api.d.ts` is **generated** by `openapi-typescript`. Never edit it by hand. Run `bun run sync:api` (requires server running) after any server API change.
- `openapi.json` is committed at `server/openapi.json` and written by `sync:api`.
- `__APP_VERSION__` is injected at build time by Vite from `package.json`. Declared as `declare const` in `src/lib/config/constants.ts` — do not use `process.env` for version.
- `shadcn-svelte` components go to `src/lib/components/ui/` (configured in `components.json`). Import via `$lib/components/ui/...`.
- API client reads `VITE_SERVER_BASE_URL`, `VITE_API_USERNAME`, `VITE_API_PASSWORD` env vars (defaults: `http://localhost:8080`, `user`, `password`).

---

## Cross-Module

- Server Spring Security uses HTTP Basic Auth with hardcoded dev credentials (`user` / `password`).
- No CI workflows exist yet.
- The artifact name in `gradle.properties` is `ilionx-pet-store`, not `vethub` — this is known and intentional.
