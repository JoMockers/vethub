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

### Domain-Specific Notes

**owner**
- `OwnerRepository.findByLastName` is an exact match — not a `LIKE` query.
- `Owner.firstName` and `Owner.lastName` intentionally lack `@NotBlank` on the entity. This is a known student exercise — do not add it to the entity; validation belongs in `OwnerValidator`.

**pet**
- The `pet` domain has no validator class. Incoming request fields are not validated beyond null-checks in the service.
- `Pet.birthDate` intentionally lacks `@Past` — future birth dates are currently accepted. This is a known student exercise.
- `PetService.delete` removes the pet from `owner.getPets()` before calling `petRepository.delete()` — this is required to avoid orphan/cascade issues given the bidirectional `@OneToMany` relationship.
- `Pet` has a `@ManyToOne` to `Owner` (FK: `owner_id`) and `PetType` (FK: `type_id`), and a `@OneToMany(cascade=ALL, fetch=EAGER)` to `Visit`.

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
- `SERVER_BASE_URL` in `constants.ts` appends `/api` to `VITE_SERVER_BASE_URL` — domain controllers use paths like `/v1/owners` directly and do not add the context path themselves.
- The `client.ts` response middleware logs non-2xx responses to the console but does **not** throw — error handling is the responsibility of each `*Controller.ts` caller.
- `bun run sync:api` runs the full pipeline: stops Gradle daemons → removes old `server/openapi.json` → boots the Spring Boot server → downloads the live spec via `curl` → runs `generate:api` → stops Gradle daemons. Requires the server to be startable.

---

## Cross-Module

- Server Spring Security uses HTTP Basic Auth with hardcoded dev credentials (`user` / `password`).
- No CI workflows exist yet.
- The artifact name in `gradle.properties` is `ilionx-pet-store`, not `vethub` — this is known and intentional.

---

## Architecture

### Overview

VetHub is a veterinary practice management application. It is a monorepo containing two independently built and run modules: a Spring Boot REST API (`server/`) and a SvelteKit single-page application (`client/`). They share no build tooling; the only coupling between them is the OpenAPI contract.

---

### Tech Stack

| Concern | Technology |
|---|---|
| Backend framework | Spring Boot 3, Java 25 |
| Build tool (server) | Gradle with Kotlin DSL |
| Database | H2 (dev/test); Liquibase for schema management |
| Object mapping | MapStruct + Lombok |
| API documentation | SpringDoc OpenAPI 3 + Swagger UI |
| Frontend framework | SvelteKit 2, Svelte 5 |
| Build tool (client) | Vite + Bun |
| HTTP client | `openapi-fetch` (type-safe fetch wrapper) |
| UI components | `shadcn-svelte` |
| Frontend types | `openapi-typescript` (generated from OpenAPI spec) |
| Auth | HTTP Basic Auth (`user` / `password` in dev) |

---

### Project Structure

```
vethub/
├── server/                        # Spring Boot API
│   └── src/main/java/dev/ilionx/workshop/
│       ├── Application.java
│       ├── api/
│       │   ├── Paths.java         # All URL constants — never hardcode paths
│       │   ├── owner/
│       │   ├── pet/
│       │   ├── vet/
│       │   └── visit/
│       └── common/
│           ├── config/            # Security, OpenAPI, Web config
│           └── exception/
├── client/                        # SvelteKit SPA
│   └── src/
│       ├── lib/
│       │   ├── api/
│       │   │   ├── client.ts      # openapi-fetch singleton
│       │   │   ├── models.ts      # re-exports generated types as named aliases
│       │   │   └── *Controller.ts # one per domain
│       │   ├── types/
│       │   │   └── api.d.ts       # GENERATED — never edit by hand
│       │   └── config/
│       │       └── constants.ts   # env vars + app version
│       └── routes/
└── scripts/                       # Cross-module shell utilities (openapi-sync.sh etc.)
```

---

### Backend Domain Package Convention

Every domain follows an identical layout. The domain is the top-level unit of organisation — not the layer.

```
api/<domain>/
  controller/    HTTP entry point; delegates to service; maps via mapper
  service/       Business logic; @Transactional; throws DataNotFoundException
  repository/    Spring Data JPA interface; derived queries only
  model/
    <Entity>.java          JPA entity
    request/               Incoming DTOs (CreateXxxRequest, UpdateXxxRequest)
    response/              Outgoing DTOs (XxxResponse)
    mapper/                MapStruct abstract class; entity ↔ DTO
    validator/             Manual field validation (owner domain only)
```

Domains: `owner`, `pet`, `vet`, `visit`.

---

### How the Layers Connect

#### Backend request flow

```
HTTP Request
  → Controller        validates (if validator exists), calls service
  → Service           loads/saves entities via repositories, throws on not-found
  → Repository        Spring Data JPA → H2 (dev) / production DB
  → Mapper            entity → response DTO
  → HTTP Response
```

#### Frontend request flow

```
Svelte component
  → *Controller.ts    calls client.GET/POST/PUT/DELETE with typed params
  → client.ts         openapi-fetch singleton; injects Basic Auth header
  → HTTP (fetch)
  → Spring Boot API
```

#### OpenAPI type sync

```
Spring Boot annotations (@Schema, @Operation, @Tag)
  → SpringDoc generates live spec at /v1/public/docs
  → bun run sync:api  → curl downloads spec → server/openapi.json (committed)
  → bun run generate:api  → openapi-typescript → src/lib/types/api.d.ts (generated)
  → models.ts         re-exports as friendly named aliases
  → *Controller.ts    consumes fully typed request/response shapes
```

Run `bun run sync:api` after any backend API change to keep the frontend types in sync.

---

### Key Patterns and Conventions

**URL constants** — all path strings live in `dev.ilionx.workshop.api.Paths`. Never hardcode a path string in a controller or test.

**MapStruct mapping** — mappers are `abstract` classes annotated with `@Mapper(config = SharedMapperConfig.class)`. Name-matched fields are wired automatically; only structural mismatches (e.g. flattening `owner.id → ownerId`) need an explicit `@Mapping`. Lombok must appear before `mapstruct-processor` in the annotation processor order — do not reorder.

**Validation** — request validation is done manually in `*Validator` components, not via Bean Validation annotations on entities. Errors are accumulated and thrown together as a `ValidationException`.

**Transactions** — read methods are `@Transactional(readOnly = true)`; write methods are `@Transactional`.

**Not-found errors** — services throw `DataNotFoundException(<ApiErrorCode>)` when a requested entity does not exist. Never return null from a service.

**Generated types** — `src/lib/types/api.d.ts` is fully generated. Never edit it. Import types via `$lib/api/models` (the re-export layer), not directly from `api.d.ts`.

**Frontend API calls** — always go through a domain `*Controller.ts` file. Never call `client.GET/POST/...` directly from a Svelte component.

**shadcn-svelte components** — add to `src/lib/components/ui/` and import via `$lib/components/ui/...`.

---

### Testing Strategy

| Type | Base class | Spring context | Subject |
|---|---|---|---|
| Unit | `UnitTest` | No | Services, validators |
| Integration | `IntegrationTest` | Yes (H2, random port) | Controllers (full HTTP stack via MockMvc) |

`UnitTest` provides static entity factory methods (`aValidOwner()`, `aValidPet()`, etc.) and enables Mockito via `@ExtendWith(MockitoExtension.class)`.

`IntegrationTest` inherits through a three-level chain: `IntegrationTest → WebMvcConfigurator → TestContextInitializer`. `TestContextInitializer` carries `@SpringBootTest(webEnvironment = RANDOM_PORT)` and `@ActiveProfiles("test")` and autowires all six repositories. `WebMvcConfigurator` builds a `MockMvc` instance in `@BeforeEach`. `IntegrationTest` adds database cleanup and provides both persistence factories (`aSavedOwner()`) and request factories (`aCreateOwnerRequest()`). Liquibase seed rows with IDs ≤ 6 (vets, pet types) and IDs ≤ 3 (specialties) are preserved — do not delete or alter those changesets.

The client has no unit tests. The only verification step is `bun run check` (svelte-check type-checking).
