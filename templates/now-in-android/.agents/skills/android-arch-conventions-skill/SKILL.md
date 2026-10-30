---
name: android-arch-conventions-skill
description: >
  Workspace layout, technology stack, SDK module patterns, repository profiles,
  local/remote dependency toggles, and test conventions for core.
  Use when: implementing features, understanding repo structure, following naming
  conventions, or working with Gradle module topology.
  Loaded by: AndroidProgrammer.
---

# Android Architecture — Conventions & Patterns

> Atomic skill. Covers workspace structure, tech stack, and implementation conventions.
> For layer/invariant rules → load `android-arch-layers-skill`.
> For build/test commands → load `android-arch-build-skill`.

---

## 1. Workspace Layout

The project follows Google's **Now in Android** multi-module architecture:

```
:app                          → Android application entry point, Application class, MainActivity, AppNavHost
build-logic/                  → Gradle convention plugins (kickoff.android.*)
core/
├── model/                    → Pure domain models and entity definitions (@Serializable)
├── common/                   → Dispatchers (@Dispatcher), Result monad (AppResult), coroutines extensions
├── connectivity/             → NetworkMonitor Flow via ConnectivityManager
├── analytics/                → AnalyticsHelper interface and logging
├── permissions/              → Compose permission handlers (rememberPermissionLauncher)
├── network/                  → Retrofit, OkHttp, Kotlinx Serialization, ApiClient
├── database/                 → Room DB, DAOs, Room entities
├── datastore/                → Preferences DataStore
├── designsystem/             → Material 3 theme (dynamic colors, typography, shapes)
└── ui/                       → Common Compose UI layouts (Loading, Error)
feature/
└── <name>/                   → Feature modules (e.g. :feature:home) with ViewModels, Compose screens, UI state, routes
```

---

## 2. Universal Technology Stack

| Dimension | Canonical Choice |
|-----------|-----------------|
| Language | Kotlin only — no new Java |
| UI framework | Jetpack Compose (new code); legacy XML views in pre-Compose modules only |
| Architecture | MVVM + Clean Architecture |
| Concurrency | Kotlin Coroutines + Flow — no RxJava in new code |
| DI | Hilt/Dagger + constructor injection (`@Inject`) |
| Build system | Gradle multi-module |
| Min API | Respect existing min SDK per repo — never lower without Lead approval |
| Testing | JUnit 5 + MockK + Turbine + kotlinx-coroutines-test |

---

## 3. Standard 4-Sub-Module SDK Pattern

Each SDK module under `services/` and `sdk-android/` follows:

```
<sdk-name>/
├── <name>-domain/           → Use case interfaces, domain models, repository contracts
├── <name>-infrastructure/   → Repository implementations, mappers, data sources, network/DB
├── <name>-presentation/     → ViewModels, Compose screens, UI state/events
└── <name>-application/      → Hilt module, DI wiring, module entry point
```

Adding or removing sub-modules from this topology is a **structural governance event** — requires an ACC registered by AndroidArchitectAgent.

---

## 4. Repository Profiles

### `common/`
- Foundation layer: core Kotlin types, domain utilities, Bluetooth/beacon abstractions, resources.
- Key modules: `atobe-core-domain`, `atobe-core-kotlin`, `atobe-core-presentation`, `atobe-geofencing`, `atobe-resources`, `atobe-versioning`.
- No dependency on `services/` or `app/`.
- Published as Maven artifacts consumed by others.

### `utilities/`
- Cross-cutting utilities: UI toolkit, map, analytics, auth, notifications, preferences, link handling.
- May depend on `common/` artifacts.
- No dependency on `services/` or `app/`.
- Published as Maven artifacts.

### `services/`
- Service SDK modules: parking, echarging, traffic, cooltra, taxi-link, transports.
- Follow the 4-sub-module SDK pattern.
- May depend on `common/` and `utilities/` artifacts.
- No dependency on `app/`.
- Published as Maven artifacts.

### `sdk-android/`
- Core SDK architecture and internals.
- May depend on `common/` artifacts.
- Published as Maven artifact consumed by `app/`.

### `app/`
- Main Android app.
- Structure: `app/`, `domain/`, `infrastructure/`, `presentation/`.
- Consumes artifacts from all other repos (or local sources during development).
- No published artifact — this is the top-level app.

---

## 5. Local vs Remote Dependency Toggles

Sibling repos can be included as:
- **Local:** module path included via `settings.gradle` (e.g., `includeBuild` or `include ':module'`)
- **Remote:** versioned Maven artifact resolved from a registry

This toggle lives in `settings.gradle` / `settingsVariables.gradle`.

**Governance rule:** Toggling a dependency source is a **structural governance event** — requires AndroidArchitectAgent ACC.

---

## 6. Test Conventions

| Test type | Tools |
|-----------|-------|
| Unit — UseCase | JUnit 5, MockK, `kotlinx-coroutines-test` |
| Unit — ViewModel | JUnit 5, MockK, turbine, `TestCoroutineScheduler` |
| Unit — Repository | JUnit 5, MockK or Fake, `kotlinx-coroutines-test` |
| Compose UI | Compose testing APIs, `createComposeRule()` |
| Integration | Depends on module; prefer Fake implementations over real infrastructure |

**Mandatory:** Every public domain interface must have a corresponding `Fake<InterfaceName>` class in `testFixtures` or test sources.
