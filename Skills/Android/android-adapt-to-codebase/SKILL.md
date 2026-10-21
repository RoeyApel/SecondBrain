---
name: android-adapt-to-codebase
description: How to decide when to follow the android-* best-practice skills literally vs. match an existing project's actual architecture, networking library, or module structure. Use this skill whenever editing or adding to an existing Android/KMP feature, before applying android-compose-ui, android-data-layer, android-di-koin, android-error-handling, android-module-structure, android-navigation, or android-presentation-mvi conventions. Trigger on phrases like "existing project", "existing feature", "legacy code", "monolith", "this codebase uses Retrofit/MVVM", "don't rewrite", "match existing patterns", or whenever the project's current architecture visibly differs from a skill's prescribed one.
---

# Adapting the Android Skills to an Existing Codebase

## Principles

- The other `android-*` skills describe a **target** architecture — MVI, Ktor, Koin, feature-layered modules — not a rule that overrides what a real project already does.
- Before applying any skill's conventions to **existing code**, identify the project's actual pattern for that concern (networking client in use, presentation style, module layout) and match it.
- Applying best practices is about the *code you are producing*, not about migrating the code around it. Don't let a small task snowball into a library swap, an architecture rewrite, or a new module.

---

## Decision Framework

| Situation | What to do |
|---|---|
| Editing inside an existing feature/file (bug fix, small addition, extending an existing ViewModel/repository) | Match the file's/feature's existing pattern exactly — same networking library, same MVVM-ish structure, same module. Do not partially convert it to MVI/Ktor/etc. as a side effect. |
| Adding a genuinely new feature/module with no existing pattern of its own | Default to the `android-*` best practices (MVI, module structure, etc.), but **keep using the project's already-chosen libraries** (e.g. Retrofit if that's the project's networking library) rather than introducing a second one for "correctness." Best practice ≠ swapping infra. |
| Explicit refactor/migration ask | The relevant skill's target applies fully — this skill's caution doesn't gate a rewrite the user actually asked for. |
| Unsure which bucket a task falls into | Ask rather than assume, especially before anything that touches more than the file(s) the user is asking about. |

---

## Anti-Patterns to Avoid

- Creating a new `core`/util module for a single helper function or a trivial addition — see `android-module-structure`'s own "don't create a separate module for a single class or trivial utility" rule; this applies doubly in a monolith that has no module split at all.
- Migrating a Retrofit call to Ktor, or changing DTO/mapper style, while fixing something unrelated in that same class.
- Restructuring a large existing ("god") ViewModel into MVI State/Action/Event as an incidental step of an unrelated feature ask.
- Introducing Koin into a project that uses a different DI approach (or none) just because a skill defaults to Koin.
- Extracting a design-system/multi-module split for a monolith project just because `android-module-structure` describes one — module structure changes are a deliberate, explicitly-requested step, not a drive-by improvement.

---

## Read Before You Write

Before writing new code in an existing project, spend a moment confirming what's actually there:

- Networking: which client is imported in neighboring repositories/data sources (Retrofit vs. Ktor vs. something else)?
- DI: how are dependencies wired today (Koin, Hilt, Dagger, manual, none)?
- Presentation: does the existing ViewModel already follow State/Action/Event, or is it ad-hoc (LiveData, mutable fields, partial MVVM)?
- Modules: is this a single-module monolith, or does it already have a `core`/`feature` split for this concern?

If the answer diverges from a skill's default, match the project — and briefly say which pattern you're matching and why, so the divergence from the skill's stated best practice is visible rather than silent.

---

## Checklist

- [ ] Identified whether this task edits existing code or adds genuinely new code
- [ ] For existing code: confirmed the project's current library/architecture for this concern and matched it
- [ ] For new code: used the project's existing libraries even if defaulting to skill best practices for structure
- [ ] No new module/library/pattern introduced solely for "correctness" outside the task's actual scope
- [ ] If a large refactor seems warranted, flagged it to the user rather than doing it silently
