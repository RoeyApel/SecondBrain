# Jetpack Compose Coding Standards

## Core Principles

- Treat composable functions as idempotent, side-effect-free, and fast: same inputs must always produce the same UI, with no mutation of external state, globals, ViewModels, or shared objects during composition.
- Assume a composable can recompose zero, one, or many times, in any order, and possibly out of order relative to sibling composables — never rely on execution order between composables.
- Assume recomposition can be skipped or cancelled/restarted before completion — never depend on a side effect emitted directly from the body of a composable to have "happened."
- Trigger all side effects (state writes, analytics, navigation, I/O) through callbacks (`onClick`, `onValueChange`) or the dedicated effect APIs, never inline in the composable body.

## Architecture & Structure

- Hoist state: a composable that needs to change a value should receive `value` and `onValueChange`-style lambdas rather than owning mutable state it can't share.
- Hoist state to the lowest common parent of all composables that read it, and no higher than the highest composable that writes it; hoist two states together if they always change in response to the same event.
- Prefer stateless composables (state passed in as parameters) for reusable/testable UI; provide a thin stateful wrapper (using `remember`) only at the point where the caller doesn't need control.
- For non-trivial screen state and logic, extract a dedicated state holder class exposed via a `rememberXState()` factory function, rather than scattering `remember` calls through the composable tree.
- Convert `Flow`, `LiveData`, and RxJava streams into Compose `State` only inside a composable (`collectAsStateWithLifecycle()` on Android, `observeAsState()`, `subscribeAsState()`), never outside composition.
- Maintain unidirectional data flow: state flows down through parameters, events flow up through callbacks.

## Code Quality & Style

- Use `remember` to survive recomposition and `rememberSaveable` to survive configuration change/process death for any state the user would expect to persist across rotation.
- Use `mapSaver`/`listSaver`/`@Parcelize` to make non-`Bundle`-safe types work with `rememberSaveable`.
- Do not duplicate a single source of truth for state at multiple levels of the tree.
- Name event callbacks specifically (`onNameChange`) rather than generically (`onValueChange`) when the composable has a specific domain meaning.
- Keep composables small and focused; extract slot-based APIs (trailing lambda content parameters) for customizable UI regions instead of adding boolean flags.

## Error Handling

- Model loading/success/error UI state explicitly (e.g., a sealed result type fed into `produceState`) rather than using nullable ad hoc flags for error conditions.
- Handle `DisposableEffect` cleanup unconditionally in `onDispose`; never leave a listener/observer registered without a matching removal.
- Do not swallow exceptions from suspend work started in `LaunchedEffect`/`rememberCoroutineScope`; surface failures into observable UI state.

## Security

- Do not log or emit sensitive values (passwords, tokens, PII) from `Text`, `SideEffect`, or debug-only composables.
- Mark screens that display sensitive content to prevent capture in recents/screenshots at the platform level; do not rely on Compose alone to protect sensitive UI.
- Do not pass secrets through composable parameters that end up in `Preview`-rendered or logged state.

## Performance

- Prefer `@Immutable`/`@Stable` data classes for parameters so Compose can skip recomposition when inputs haven't changed; avoid passing unstable types (plain `List`, lambdas capturing changing state) where a stable alternative exists.
- Always provide a stable, unique `key` in `items()` for `LazyColumn`/`LazyRow`/`LazyGrid` to avoid unnecessary recomposition and preserve item identity across list changes.
- Use `derivedStateOf` only when an input changes more frequently than the UI actually needs to recompose (e.g., collapsing a rapidly-changing scroll offset into a boolean threshold) — do not use it to merge two states that change at the same rate; combine those directly.
- Defer reads of frequently-changing state into lambda-based modifiers (e.g., `Modifier.offset { IntOffset(...) }`) instead of reading the state value directly in the composable body, to keep recomposition scoped to layout/draw instead of composition.
- Cache expensive computations with `remember(keys...)`; never perform expensive calculation, file I/O, or SharedPreferences access directly in the composable body.
- Never write to state that was already read earlier in the same composable/frame ("backwards write") — it can cause infinite recomposition loops.
- Build and profile in release mode with R8 enabled; debug-mode Compose has significant overhead and is not representative of real performance.
- Use Baseline Profiles for release builds to reduce jank on cold/warm start and during key interactions.

## Testing

- Test through semantics (`onNodeWithText`, `onNodeWithContentDescription`, `onNodeWithTag`), not implementation details.
- Add explicit `Modifier.testTag(...)` for elements that lack a stable, unique semantic property to select on.
- Rely on Compose's built-in idling/synchronization; never use `Thread.sleep` to wait for recomposition or animations in tests.
- Use `createComposeRule()` for isolated composable tests and `createAndroidComposeRule<Activity>()` when Activity/lifecycle context is required.
- Prefer stateless composables in tests where possible — pass state and callbacks directly rather than depending on ViewModels/DI in unit-level Compose tests.

## Anti-Patterns to Avoid

- Mutating a captured local variable (e.g., a counter) inside a composable body as a substitute for derived state.
- Using `LaunchedEffect(true)` / `LaunchedEffect(Unit)` when the effect should actually restart on a changing key — treat an unkeyed `LaunchedEffect` as a signal to double check.
- Reading high-frequency state (scroll position, drag offset) directly in the composable body when a lambda-deferred modifier would avoid recomposition.
- Storing mutable, non-observable collections (`ArrayList`, mutable data class) as UI state — use immutable collections wrapped in `State`.
- Leaving `DisposableEffect` without meaningful cleanup, or omitting `onDispose` entirely.
- Overusing `derivedStateOf` for cheap, same-rate computations where it adds overhead without reducing recomposition.

## Sources

- [Thinking in Compose / Mental model](https://developer.android.com/jetpack/compose/mental-model)
- [State and Jetpack Compose](https://developer.android.com/jetpack/compose/state)
- [Side-effects in Compose](https://developer.android.com/develop/ui/compose/side-effects)
- [Compose performance](https://developer.android.com/jetpack/compose/performance)
- [Compose testing](https://developer.android.com/jetpack/compose/testing)
