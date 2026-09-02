# Knowledge Contract — adempiere-jwt-token

## 1. Identity

| Field | Value |
|---|---|
| Name | adempiere-jwt-token |
| Repository Type | Library |
| Classification basis | Declared in `.ai/repository.yml` (`type: Library`) |
| Standards | knowledge-contract-v1, repository-classification-v1 |
| Component type | Java library / ADempiere token generator |
| Language and target | Java 17 (`sourceCompatibility = 1.17` in `build.gradle`) |
| Build / runtime | Gradle wrapper 7.3.3 (`gradle/wrapper/gradle-wrapper.properties`); Gradle plugins `java-library`, `maven-publish`, `signing` |
| Published artifact | `io.github.adempiere:adempiere-jwt-token` (version from `ADEMPIERE_LIBRARY_VERSION`, default `local-1.0.0`) |
| Version | `baseVersion = 3.9.4`; observed tag `adempiere-3.9.4-1.0.4` |
| License | GNU General Public License v2 |
| Root package or module | `org.spin.eca52` |
| Owner | ERP Consultores y Asociados |
| Entity type | ECA52 |
| Forked from upstream | Declared in `.ai/repository.yml`; exact upstream URL masked in evidence, recorded in UNKNOWN |

---

## 2. Responsibility

This repository is a reusable library that generates and validates JWT tokens for ADempiere third-party access.

**What it owns**

- The JWT generation and validation implementation that implements ADempiere's `IThirdPartyAccessGenerator`: `org.spin.eca52.security.JWT`.
- The setup process `org.spin.eca52.setup.CreateTokenDefinition` that creates a JWT token definition and a system configurator entry.
- Constants shared by the implementation: `org.spin.eca52.util.JWTUtil`.
- XML migrations that create this repository's own dictionary records: `AD_EntityType` ECA52, `AD_SetupDefinition`, and `AD_Message`.
- The published artifact contract `io.github.adempiere:adempiere-jwt-token`.

**What it does not own**

- The ADempiere core security system, `MADToken`, `MADTokenDefinition`, or `IThirdPartyAccessGenerator` — those come from the `io.github.adempiere:base` dependency.
- The actual JWT secret value. It creates the `ECA52_JWT_SECRET_KEY` system configurator with an empty value; a deployment must supply the secret.
- The external service or middleware consuming the generated token.

---

## 3. Architecture

```text
src/main/java/org/spin/eca52/
├── security/JWT.java                 # IThirdPartyAccessGenerator implementation
├── setup/CreateTokenDefinition.java  # ISetupDefinition implementation
└── util/JWTUtil.java                 # ECA52 constants

xml/migration/
├── 10300_ECA52_Add_Entity_Type.xml
├── 10310_ECA52_Add_Setup_for_Token_Definition.xml
└── 10320_ECA52_Add_Error_Message.xml
```

**Patterns and extension points**

- `JWT` builds JWTs with `io.jsonwebtoken.Jwts.builder()` and verifies them with `Jwts.parser().verifyWith(...)`.
- The signing key is read at runtime from `MSysConfig` using `JWTUtil.ECA52_JWT_SECRET_KEY` and the current client ID, then Base64-decoded.
- Generated JWTs carry claims `AD_Client_ID`, `AD_Org_ID`, `AD_Role_ID`, `AD_User_ID`, `M_Warehouse_ID`, and `AD_Language`.
- Expiration is optional, controlled by `MADTokenDefinition.isHasExpireDate()` and `getExpirationTime()`.
- `CreateTokenDefinition` creates one `MADTokenDefinition` with value `JWT` and one `MSysConfig` with name `ECA52_JWT_SECRET_KEY`, leaving the value empty.
- All dictionary changes are written as XML migrations under `xml/migration`.

### Pre-existing records this repository modifies

| Record | Table | Columns changed | Effect | Migration |
|---|---|---|---|---|
| None | — | — | — | — |

The evidence reports `UPDATES TO PRE-EXISTING RECORDS: None`.

---

## 4. Dependencies

| Dependency | Scope | Purpose |
|---|---|---|
| `io.github.adempiere:base:3.9.4` | build/runtime | ADempiere core classes: `org.compiere.model.*`, `org.spin.model.*`, `org.spin.util.*` interfaces implemented by this library |
| `io.jsonwebtoken:jjwt-*` | build/runtime | JWT creation and verification (`Jwts`, `JwtBuilder`, `JwtParser`, `security.Keys`). Exact coordinates/versions masked in evidence; see UNKNOWN |
| `com.fasterxml.jackson.databind.ObjectMapper` | runtime | Parses the JWT payload in `JWT.validateToken`. Used in source; likely transitive through jjwt/base rather than directly declared in the visible build file |
| Gradle wrapper `gradle-7.3.3-bin.zip` | build | Pinned distribution in `gradle/wrapper/gradle-wrapper.properties` |
| JDK 17 | build/runtime | Required by `sourceCompatibility = 1.17` and CI setup |

No secrets, credentials, tokens, or sensitive configuration values are listed.

---

## 5. Consumers

- **ADempiere installations and middleware** consume the published artifact `io.github.adempiere:adempiere-jwt-token` to generate or validate third-party access tokens.
- **ADempiere setup process** consumes `org.spin.eca52.setup.CreateTokenDefinition` through `ISetupDefinition`, creating a JWT token definition and the `ECA52_JWT_SECRET_KEY` system configurator.
- **Breaking surfaces**
  - The implemented `IThirdPartyAccessGenerator` methods `generateToken(int, int)` and `validateToken(String)`.
  - The SysConfig key `ECA52_JWT_SECRET_KEY`; renaming it breaks deployed configurations.
  - The JWT claims consumers rely on.
  - Dictionary records this repository creates: `AD_SetupDefinition` Record_ID 50079, `AD_Message` Record_ID 53757, entity type ECA52.
  - Package names `org.spin.eca52.security`, `org.spin.eca52.setup`, `org.spin.eca52.util`.

---

## 6. Allowed changes

- Modify JWT generation/validation logic inside `org.spin.eca52.security.JWT` while preserving the `IThirdPartyAccessGenerator` interface and the token format.
- Add or update XML migrations in `xml/migration` that insert new records under entity type `ECA52`.
- Update `io.jsonwebtoken` dependency versions, as already done in commit `f81bedb` (`feat: Update io.jsonwebtoken:jjwt dependencies.`), provided the JWT API behavior remains compatible.
- Adjust publication configuration in `build.gradle` so releases publish to this fork's own package repository, as described in the build file comments.
- Change the build-time version through the `ADEMPIERE_LIBRARY_VERSION` environment variable on release.
- Add automated tests for token generation and validation.

---

## 7. Prohibited changes

- Do not embed a value for `ECA52_JWT_SECRET_KEY` in source code, migrations, or build files; the secret is supplied by the deployment and read from `MSysConfig` at runtime.
- Do not modify pre-existing Base/Patch dictionary records from this library; the evidence shows none are currently modified.
- Do not rename `ECA52_JWT_SECRET_KEY` or the entity type `ECA52` without coordinated migrations and explicit consumer changes.
- Do not change the published coordinates `io.github.adempiere:adempiere-jwt-token` casually; the same coordinate exists in central upstream, so publication must remain deliberately scoped to this fork's packages or be handled with an explicit repository decision.
- Do not add UI, service, or central ERP business logic to this library.
- Do not track additional build output or IDE metadata: `.gradle/`, `build/`, `.idea/`. Already-tracked `.classpath`, `.project`, and `.settings/` should be removed rather than extended.

---

## 8. Architectural rules

1. Java code stays under `org.spin.eca52.*`.
2. Dictionary changes stay in `xml/migration`, use entity type `ECA52`, and insert only records this repository owns.
3. The JWT secret must always be read at runtime from `MSysConfig` and must never appear as a literal in code or migration.
4. The artifact remains `io.github.adempiere:adempiere-jwt-token`, with version set by `ADEMPIERE_LIBRARY_VERSION` at build time.
5. The JWT payload claims (`AD_Client_ID`, `AD_Org_ID`, `AD_Role_ID`, `AD_User_ID`, `M_Warehouse_ID`, `AD_Language`) are part of the integration contract and may be changed only through a controlled, versioned change.
6. The public integration points are the `IThirdPartyAccessGenerator` implementation in `JWT`, the `ISetupDefinition` implementation in `CreateTokenDefinition`, and the constants in `JWTUtil`.

---

## 9. Risks

| Check | Finding | Impact | Precaution |
|---|---|---|---|
| Identifiers outside the allowed allocation range | No allocation range is declared in this repository's evidence. Observed created records use IDs 50079, 50153, 53757, and `AD_Message_Trl` ID 0. Out-of-range status cannot be assessed. | Cannot detect an allocation conflict with other repositories sharing the ADempiere dictionary. | Declare the ECA52 identifier allocation range in `.ai/repository.yml` or this contract. |
| Build output or IDE metadata under version control | `.classpath`, `.project`, and `.settings/` are tracked by git. | Machine-specific files create checkout-dirty behavior and diff noise; local IDE state can leak into version control. | Remove them from tracking and add them to `.gitignore`. |
| Secrets in the tree or recoverable from history | None found. Workflow and build files reference secrets through environment variables or GitHub contexts, but no actual secret value is present in the evidence. Masked values are collector masking by key name, not evidence of a committed secret. | N/A | Keep secrets in CI/GitHub secret stores; never commit real values. |
| Absent verification mechanism | No test source tree or test dependencies are visible; CI runs `./gradlew build`, which verifies compilation but no tests. | Logic changes are verified only by compilation or manual release-candidate testing. | Add automated tests; keep using the `erp-ai:candidate` / `erp-ai:verified` workflow for manual verification when tests are absent. |
| Pre-existing records modified (cross-reference section 3) | None found. | N/A | Preserve the inserts-only discipline for dictionary migrations. |

| Risk | Impact | Precaution |
|---|---|---|
| Same Maven coordinate as upstream central artifact | Consumers resolving `io.github.adempiere:adempiere-jwt-token` may receive upstream's artifact rather than this fork's build if repository resolution is wrong. | Publish only to the fork's package repository and document the correct repository for consumers. |
| Empty default `ECA52_JWT_SECRET_KEY` | Until a deployment fills the SysConfig value, token generation fails with `@ECA52_JWT_SECRET_KEY@ @NotFound@`. | Document the setup step; verify the secret is set before relying on token generation. |
| Gradle version mismatch | README says Gradle 8.0.1 or later; the wrapper pins Gradle 7.3.3. | Builds may behave differently from the documented toolchain. | Align README and wrapper. |
| JDK mismatch for consumers | README says JDK 11 or later, but `build.gradle` targets Java 17 and CI uses JDK 17. | Consumers on JDK 11 may not load the compiled library. | Confirm consumer environments or align README with actual target. |
| Swallowed exception in token validation | In `JWT.validateToken`, the catch block constructs `new AdempiereException(e)` but does not throw it; parsing/verification failures can continue execution with a null encrypted value. | Invalid or malformed tokens may not fail fast, making validation failures hard to diagnose. | Fix the catch block to throw or return false, and add validation tests. |

---

## 10. Current state

- The repository has a declared marker: `type: Library`, `entityType: ECA52`, owner `ERP Consultores y Asociados`.
- It builds with Java 17 and Gradle wrapper 7.3.3.
- It provides three source classes: `org.spin.eca52.security.JWT`, `org.spin.eca52.setup.CreateTokenDefinition`, and `org.spin.eca52.util.JWTUtil`.
- XML migrations create:
  - `AD_EntityType` ECA52, Record_ID 50153.
  - `AD_SetupDefinition`, Record_ID 50079.
  - `AD_Message`, Record_ID 53757, with `AD_Message_Trl` insertions/updates against Record_ID 0.
- No pre-existing records are modified.
- The repository has no visible test suite.
- CI compiles with `./gradlew build`; a release-candidate workflow exists for publishing PR candidates and needing manual verification.
- A tag `adempiere-3.9.4-1.0.4` exists.
- `.classpath`, `.project`, and `.settings/` are tracked under version control.

---

## 11. UNKNOWN

- Exact upstream URL. It is declared in `.ai/repository.yml` under `upstream=`, but the evidence collector masked the value. Verify in `.ai/repository.yml`.
- Exact `io.jsonwebtoken` dependency coordinates and versions. The build.gradle lines are masked; verify in `build.gradle`.
- Exact values for masked fragments in `build.gradle`, `settings.gradle`, and workflow variable names: `artifactId`, `rootProject.name`, publication URLs, and several env variable assignments. Verify in the repository.
- The ECA52 identifier allocation range. No range is declared in evidence; verify with the repository owner or allocation registry.
- Intended Gradle version. README says 8.0.1 or later, wrapper pins 7.3.3. Verify which one is intended for this fork.
- The currently published artifact versions. Version is built from `ADEMPIERE_LIBRARY_VERSION`; the evidence shows a tag but not every published version. Verify in package repository/GitHub releases.