---
name: android-navigation
description: Type-safe Navigation3 (androidx.navigation3) for Android/KMP - NavKey route objects, per-feature entry-provider factories, a single global NavDisplay/backStack owned in :app, and cross-feature callbacks. Use this skill whenever setting up navigation, defining routes, adding a new screen to an entry provider, navigating between features, or wiring the global NavDisplay in the app module. Trigger on phrases like "set up navigation", "add a route", "navigate between screens", "NavDisplay", "entryProvider", "NavKey", "backstack", "rememberNavBackStack", "type-safe nav", or "cross-feature navigation". See android-adapt-to-codebase before migrating an existing navigation setup to Nav3.
---
 
# Android / KMP Navigation
 
## Principles
 
- If this project's navigation isn't already Nav3/NavKey-based, see `android-adapt-to-codebase` before migrating it as a side effect of an unrelated task.
- **Type-safe navigation** with `NavKey` route objects/data classes — `@Serializable` by default; required only where backstack process-death persistence matters (see Route Objects).
- **One global `backStack`** (`SnapshotStateList<NavKey>`) and **one global `NavDisplay`**, owned in `:app` (or a thin `core:presentation:navigation` module) — not one per feature.
- **One entry-provider factory function per feature**, defined in the feature's `presentation` module, mirroring today's per-feature graph split without owning any navigation state.
- Navigation is just **list mutation**: `navigate = { backStack.add(it) }`, `onBack = { backStack.removeAt(backStack.lastIndex) }` — no `NavController`.
- Feature-to-feature navigation uses **callbacks only** — never a direct route import across features.
 
---
 
## Route Objects
 
Define routes as plain `NavKey` implementations in the feature's `presentation` module. A `NavKey` only needs to be stable/equatable — `@Serializable` is not required by the framework, but is required for the route to survive process death via `rememberNavBackStack`'s saved-state mechanism.
 
```kotlin
// feature:notes:presentation
sealed interface NotesRoute : NavKey {
    @Serializable data object NoteList : NotesRoute
    @Serializable data class NoteDetail(val noteId: String) : NotesRoute
}
```
 
Group a feature's routes under a per-feature sealed interface extending `NavKey` (e.g. `NotesRoute`) instead of flat top-level objects — this keeps nested routes organized without any feature importing another feature's routes.
 
Default to `@Serializable` on every route unless there's a specific reason not to (e.g. a deliberately transient, never-revisited destination).
 
**Default to passing IDs, not whole domain objects, as route arguments.** This is not a serialization limitation — a `@Serializable` domain object works perfectly well as a route argument and Nav3 will persist it across process death like any other field. The reason to prefer IDs is architectural: an object baked into a route is a point-in-time snapshot that goes stale if the underlying data changes elsewhere, it adds payload weight to the backstack's saved state, and in a multi-module setup it forces the route (and therefore any feature that navigates to it) to depend on the domain model type.
 
```kotlin
// Default — no dependency on the domain model, always fresh when loaded
@Serializable data class NoteDetail(val noteId: String) : NotesRoute
```
 
Passing a `@Serializable` object directly is an accepted exception when the object is small, stable, and the coupling is acceptable — e.g. navigating within a single feature, or in a single-module app where every route already shares the same domain types:
 
```kotlin
// Accepted exception — Note is small/stable and this stays within one feature module
@Serializable data class NoteDetail(val note: Note) : NotesRoute
```
 
---
 
## Feature Entry Provider
 
Each feature exposes an `EntryProviderBuilder<NavKey>` extension function — the Nav3 equivalent of today's `NavGraphBuilder.<feature>Graph(...)`:
 
```kotlin
// feature:notes:presentation
fun EntryProviderBuilder<NavKey>.notesEntries(
    navigate: (NavKey) -> Unit,
    onBack: () -> Unit,
    onNavigateToEditor: (String) -> Unit  // callback for cross-feature navigation
) {
    entry<NotesRoute.NoteList> {
        NoteListRoot(
            onNavigateToDetail = { navigate(NotesRoute.NoteDetail(it)) }
        )
    }
    entry<NotesRoute.NoteDetail> { route ->
        NoteDetailRoot(
            noteId = route.noteId,
            onNavigateToEditor = onNavigateToEditor,
            onBack = onBack
        )
    }
}
```
 
The factory receives `navigate`/`onBack` lambdas rather than a `NavController` — it never touches the backstack directly, and it never receives the whole `backStack` list (that stays owned in `:app`).
 
---
 
## Wiring in `:app`
 
The backstack, decorators, and `NavDisplay` are all owned in **one place** — `:app` (or a shared `core:presentation:navigation` module if multiple entry points need it). This centralization matters more, not less, in a multi-module setup: decorators and transition specs must not be duplicated per feature.
 
```kotlin
// :app
@Composable
fun AppNavGraph() {
    val backStack = rememberNavBackStack(NotesRoute.NoteList)
    val navigate: (NavKey) -> Unit = { backStack.add(it) }
    val onBack: () -> Unit = { backStack.removeAt(backStack.lastIndex) }
 
    NavDisplay(
        backStack = backStack,
        entryDecorators = listOf(
            rememberSaveableStateHolderNavEntryDecorator(),
            rememberViewModelStoreNavEntryDecorator()
        ),
        transitionSpec = NavDefaults.transitionSpec,
        popTransitionSpec = NavDefaults.popTransitionSpec,
        entryProvider = entryProvider {
            notesEntries(
                navigate = navigate,
                onBack = onBack,
                onNavigateToEditor = { navigate(EditorRoute.Edit(it)) }
            )
            editorEntries(navigate = navigate, onBack = onBack)
        }
    )
}
```
 
Cross-feature navigation is always expressed as a lambda callback passed into the feature's entry-provider factory — never by importing a route from another feature module.
 
Define shared transition specs once (e.g. `NavDefaults` in `core:design-system` or `core:presentation:navigation`) rather than inlining `transitionSpec`/`popTransitionSpec` literals at the `NavDisplay` call site. A single global `NavDisplay` mostly avoids duplication by construction, but naming the defaults object keeps intent explicit if a second `NavDisplay` (e.g. for a dialog-hosted sub-flow) is ever introduced.
 
---
 
## Entry Decorators & ViewModel Scoping
 
`entryDecorators` replaces behavior Nav2 provided implicitly:
 
- `rememberSaveableStateHolderNavEntryDecorator()` — preserves `rememberSaveable` state across an entry being pushed/popped.
- `rememberViewModelStoreNavEntryDecorator()` — scopes each entry's `ViewModelStore` to that backstack entry, so a ViewModel is cleared when its entry is removed from `backStack`.
 
Because ViewModel lifetime is now tied to backstack entry membership, **do not manually retain a ViewModel elsewhere** (e.g. in a Koin singleton scope or a static holder) expecting it to outlive its entry — doing so defeats the decorator's scoping and leaks the ViewModel past its screen's lifetime. If state must genuinely outlive a screen, promote it to a shared use case/repository instead of holding onto the ViewModel.
 
Keep the decorator list defined once, in the same place `NavDisplay` is called (`:app` / `core:presentation:navigation`) — never per feature.
 
---
 
## Backstack & Process Death
 
`rememberNavBackStack(startRoute)` returns a `SnapshotStateList<NavKey>` that the app owns and mutates directly. It survives configuration changes automatically. It survives **process death** only if every `NavKey` currently on the stack is `@Serializable` — a non-serializable route on the stack at the time of process death will fail to restore.
 
This matters most for deep or rarely-revisited stacks (e.g. a multi-step form flow) — favor `@Serializable` routes by default per Route Objects above, and treat any deliberately non-serializable route as one the user is expected to be routed back out of before the process is likely to die.
 
---
 
## Testing
 
Because `backStack` is a plain `SnapshotStateList<NavKey>` rather than an opaque `NavController`, navigation logic can be asserted on directly in tests — e.g. asserting `backStack.last() == NotesRoute.NoteDetail("abc123")` after invoking a callback, with no `NavController`/Robolectric harness required. See the `android-testing` skill for this project's general ViewModel/Compose testing conventions; this is a Nav3-specific win worth applying wherever navigation callbacks are exercised in a test.
 
---
 
## Naming Conventions
 
| Thing | Convention | Example |
|---|---|---|
| Nav route | `<Screen>Route` under a per-feature `<Feature>Route` sealed interface | `NotesRoute.NoteList`, `NotesRoute.NoteDetail` |
| Feature entry provider | `<feature>Entries(...)` on `EntryProviderBuilder<NavKey>` | `notesEntries(...)` |
| Global nav graph composable | `AppNavGraph` (or similar), owned in `:app` | `AppNavGraph()` |
| Shared transition defaults | `NavDefaults` | `NavDefaults.transitionSpec` |
 
---
 
## Checklist: Adding Navigation to a New Feature
 
- [ ] Define `NavKey` routes under a per-feature sealed interface in `feature:presentation` (`@Serializable` by default)
- [ ] Add a feature entry-provider function (`EntryProviderBuilder<NavKey>.<feature>Entries(...)`)
- [ ] Pass `navigate`/`onBack` lambdas into the factory for intra-feature navigation — never the raw `backStack`
- [ ] Expose cross-feature destinations as lambda callbacks (not direct route imports)
- [ ] Wire the feature's `entries` function into the single global `entryProvider { }` block in `:app`
- [ ] Confirm no route passes a whole domain object as an argument unless it's a deliberate, scoped exception
