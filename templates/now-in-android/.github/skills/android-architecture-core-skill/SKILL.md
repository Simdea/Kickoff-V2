---
name: android-architecture-core-skill
description: >
  Index of the Android architecture knowledge base. Do not load this skill
  directly — load the specific atomic skill your role requires (see table below).
  Loaded by: no agent directly — agents load atomic sub-skills instead.
---

# Android Architecture Core — Index

This skill has been split into focused atomic skills to minimize context window usage.
Load only the skill your role requires.

---

## Skill Map

| Skill | Contents | Who loads it |
|-------|----------|-------------|
| `android-arch-layers-skill` | Layer architecture, dependency direction, DI rules, Coroutines/Flow contracts, Non-negotiable invariants | AndroidArchitectAgent, AndroidTestReviewer, LeadAndroidAgent |
| `android-arch-conventions-skill` | Workspace layout, tech stack, SDK module patterns, repo profiles, local/remote toggles, test conventions | AndroidProgrammer |
| `android-arch-build-skill` | Build commands, test execution commands, Gradle task reference | AndroidProgrammer |
| `android-architect-validation-skill` | Scope classification, verdict protocol, ACC/ACC-lite templates | AndroidArchitectAgent |
| `android-implementation-skill` | Implementation patterns, UseCase/Repository/ViewModel scaffolding | AndroidProgrammer |
| `android-test-chaos-skill` | Adversarial test checks, check matrices, ACCEPTABLE/INSUFFICIENT protocol | AndroidTestReviewer |
| `android-lead-orchestration-skill` | Delegation rules, gate sequencing, ACC registration | LeadAndroidAgent |

---

## Workspace Repos

| Repo | Purpose |
|------|---------|
| `app` | Main multi-service app (Compose + Hilt + MVVM) |
| `sdk-android` | Core SDK (published artifact) |
| `services` | Feature SDKs (parking, charging, transit…) |
| `utilities` | Cross-cutting SDKs (auth, analytics, core-domain…) |
| `common` | Shared modules: core-kotlin, resources, geofencing, versioning |

---

## Non-Negotiable Invariants (quick reference)

> Full rules live in `android-arch-layers-skill`.

- **Hilt** is the canonical DI framework. No ServiceLocator, no manual injection.
- **StateFlow** is the standard UI state contract. ViewModels expose `StateFlow<UiState>`.
- **GlobalScope** is prohibited. Always use structured concurrency scoped to a lifecycle.
- **runBlocking** is prohibited in production code. Use `suspend` + `CoroutineScope`.
- **Dispatchers** must be injected — never hardcode `Dispatchers.IO` or `Dispatchers.Main`.
- Domain module must be framework-free (no Android/Hilt/Room imports).

---

## Why atomic skills?

Each agent loads only the knowledge quantum it needs:
- `AndroidArchitectAgent` needs layer rules and invariants — not build commands.
- `AndroidProgrammer` needs conventions and build commands — not verdict protocols.
- `AndroidTestReviewer` needs layer invariants — not workspace layout.

Loading the full 340-line monolith per invocation wastes 200–250 tokens of context
on information that is irrelevant to the agent's immediate task.
