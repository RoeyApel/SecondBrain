---
name: android-compose-ui
description: Compose UI patterns for Android/KMP - stability, recomposition, side effects, lazy lists, animations, previews, accessibility, modifier extensions, and design system composables. Use this skill whenever writing or reviewing composables, optimizing recomposition, adding animations, creating previews, writing custom modifiers, structuring a design system, or making any Compose UI decision beyond the MVI/ViewModel layer. Trigger on phrases like "composable", "recomposition", "LaunchedEffect", "Modifier", "LazyColumn", "preview", "animation", "design system", "stability", "contentDescription", "graphicsLayer", "slot API", or "Compose performance". See android-adapt-to-codebase before rewriting existing screens/composables to match these patterns wholesale.
---

# Android / KMP Compose UI Patterns

## Core Principle

The UI is dumb. Composables render state and forward user actions — nothing more. All state lives in the ViewModel. All logic lives in the ViewModel, domain, or data layer. Compose code should contain zero business logic, zero data transformation, and minimal side effects.

If existing composables already mix in business logic or don't follow this split, see `android-adapt-to-codebase` before refactoring them as a side effect of an unrelated task.

---

## Stability & Recomposition

Strong skipping mode is enabled by default in modern Compose — no explicit opt-in needed.

Only annotate a state data class with `@Stable` when it contains fields the Compose compiler considers unstable (e.g., `List`, `Map`, `Set`, interfaces, or abstract types). If all fields are primitive types, `String`, or other stable types, no annotation is needed.

```kotlin
// Needs @Stable — contains a List (unstable by default)
@Stable
data class NoteListState(
    val notes: List<NoteUi> = emptyList(),
    val isLoading: Boolean = false
)

// No annotation needed — all fields are stable
data class NoteDetailState(
    val title: String = "",
    val body: String = "",
    val isSaving: Boolean = false
)
```

---

## State Ownership

All state lives in the ViewModel. Do not use `remember` or `rememberSaveable` for application state — that belongs in the ViewModel's `StateFlow` and is surfaced via `collectAsStateWithLifecycle()`.

The only exception is Compose-internal state that the framework requires you to hold in composition, such as `LazyListState`, `ScrollState`, or `PagerState`. For these, use `remember*` as needed:

```kotlin
// Acceptable — Compose-owned UI state
val lazyListState = rememberLazyListState()

// Reacting to Compose-owned state with derivedStateOf
val showScrollToTop by remember {
    derivedStateOf { lazyListState.firstVisibleItemIndex > 5 }
}
```

`derivedStateOf` should only be used in these rare scenarios where Compose-internal state drives a derived value. If the derivation can happen in the ViewModel, it should.

Always collect ViewModel state with lifecycle awareness:
```kotlin
val state by viewModel.state.collectAsStateWithLifecycle()
```

---

## Side Effects

Side effects should be avoided when possible. If something can be handled by the ViewModel through an Action, do that instead of using a side effect in a composable.

When a side effect is truly necessary (e.g., interacting with Android lifecycle APIs that have no ViewModel equivalent), extract it into a dedicated composable to keep the Screen composable clean:

```kotlin
// Extracted into its own composable
@Composable
fun ObserveLifecycle(onStart: () -> Unit, onStop: () -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_START -> onStart()
                Lifecycle.Event.ON_STOP -> onStop()
                else -> Unit
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }
}
```

`LaunchedEffect` is acceptable when genuinely needed, but question whether the work belongs in the ViewModel first. Do not use custom `CompositionLocal`s.

---

## Lazy Layouts

Add `key` to lazy list items when there is an obvious unique identifier available. Don't force it if it's unclear which property is unique:

```kotlin
LazyColumn {
    items(
        items = state.notes,
        key = { it.id }  // id is clearly unique
    ) { note ->
        NoteItem(
            note = note,
            onClick = { onAction(OnNoteClick(note.id)) },
            modifier = Modifier.tag(NoteListTestTags.noteItem(note.id))
        )
    }
}
```

The same id that drives `key` also drives the test tag — see **Test Tags** below.

---

## Animations

Avoid animations that cause recompositions. Prefer approaches that animate below the recomposition layer:

- **`graphicsLayer`** — for alpha, scale, rotation, translation
- **Offset lambda** — for position changes (`offset { ... }`)
- **`Canvas`** — for custom drawing that animates
- **`animateFloatAsState` + `graphicsLayer`** — animate a float, apply in graphicsLayer

```kotlin
// Good — animates without recomposition
val alpha by animateFloatAsState(if (state.isVisible) 1f else 0f)
Box(
    modifier = Modifier.graphicsLayer { this.alpha = alpha }
)

// Bad — causes recomposition on every frame
Box(
    modifier = Modifier.alpha(animatedAlpha)
)
```

**Deferred state reads:** When a value drives an animation, pass it as a lambda rather than reading it directly. This defers the state read to the layout/draw phase and avoids recomposition:

```kotlin
// Good — deferred read
fun Modifier.animatedOffset(offsetProvider: () -> IntOffset) =
    offset { offsetProvider() }

// Bad — immediate read causes recomposition
fun Modifier.animatedOffset(offset: IntOffset) =
    offset(x = offset.x.dp, y = offset.y.dp)
```

---

## Modifier Extensions

Prefer plain `Modifier` extension functions or `Modifier.Node`-based factories. Do not make modifier extensions `@Composable`:

```kotlin
// Good — plain extension
fun Modifier.shimmerEffect(): Modifier = composed {
    // shimmer implementation
}

// Better — Modifier factory (no composition needed)
fun Modifier.roundedBackground(color: Color, radius: Dp) =
    background(color, RoundedCornerShape(radius))
```

---

## Design System & Slot APIs

The design system lives in `:core:design-system` and contains reusable Compose components, colors, theme, and typography.

Use slot APIs (passing `@Composable` lambdas) primarily for design system components that need flexible content areas:

```kotlin
// Slot API — design system component
@Composable
fun AppCard(
    modifier: Modifier = Modifier,
    header: @Composable () -> Unit,
    content: @Composable () -> Unit
) {
    Card(modifier = modifier) {
        header()
        content()
    }
}
```

Feature-level composables should prefer typed parameters over slots for clarity.

---

## Previews

Every Screen composable should have at least one meaningful `@Preview` that shows a realistic state:

```kotlin
@Preview
@Composable
private fun NoteListScreenPreview() {
    AppTheme {
        NoteListScreen(
            state = NoteListState(
                notes = listOf(
                    NoteUi("1", "Meeting notes", "Mar 15"),
                    NoteUi("2", "Shopping list", "Mar 14")
                )
            ),
            onAction = {}
        )
    }
}
```

Wrap previews in the app theme so they reflect real appearance. Use realistic sample data, not empty states (unless previewing the empty state specifically).

---

## Accessibility

Use meaningful `contentDescription` on all interactive or informational visual elements. Always use string resources to allow localization:

```kotlin
Icon(
    imageVector = Icons.Default.Delete,
    contentDescription = stringResource(R.string.cd_delete_note)
)

Image(
    painter = painterResource(R.drawable.profile),
    contentDescription = stringResource(R.string.cd_profile_picture)
)
```

For decorative elements that convey no information, set `contentDescription = null`.

---

## Test Tags

`testTag` and `contentDescription` are orthogonal. `contentDescription` is for accessibility — it's read by screen readers. `testTag` is invisible to accessibility services and exists purely so tests and automation tools (Espresso, Maestro) can find a node. A node can carry both.

### Shared Helpers (declared once)

Test code has no `Modifier` in scope, but it needs to build the exact same runtime string production code applies — so the prefix-joining logic is a plain function that both a `Modifier` extension and test code call into. Declare both once in `:core:design-system` (or any similar module if it doesn't exist):

```kotlin
// :core:design-system — declared once, used by production code AND tests
fun buildTestTag(prefix: String, tag: String): String = "${prefix}_$tag"  
  
fun Modifier.testTag(prefix: String?, tag: String): Modifier =  
    if (prefix == null) this else this.testTag(buildTestTag(prefix, tag))
```

The `Modifier` overload accepts a nullable tag and no-ops when it's `null` (for elements whose tag depends on optional data) rather than ever emitting a `"prefix_null"` string.

### Per-Screen Convention

Each screen defines an `object <Screen>TestTags` in the same file as its Screen composable. It holds a public `PREFIX` (test code needs it too, so it can't be private) plus one entry per element:

- A `const val` for static, one-off elements.
- A function taking the unique id for repeated elements (list rows, per-item buttons) — the id is baked into the tag so every rendered instance is distinctly, directly matchable. This mirrors the existing `key = { it.id }` convention from **Lazy Layouts**.

Alongside the object, declare a screen-local `Modifier.tag(...)` wrapper that closes over `PREFIX`, so call sites only ever pass the short tag. It's declared in the same file it's used in, so there's no import to disambiguate between screens — every screen can reuse the same short name, `tag`.

```kotlin
object NoteListTestTags {
    const val PREFIX = "note_list"
    const val LOADING_INDICATOR = "loading_indicator"
    const val EMPTY_STATE = "empty_state"
    const val REFRESH_BUTTON = "refresh_button"
    const val DELETE_NOTE_BUTTON = "delete_note_button"
    fun noteItem(id: String) = "note_item_$id"
}

// Screen-local wrapper around the shared extension — closes over this screen's prefix.
// Named `tag` in every screen file; no import needed since it's declared where it's used.
fun Modifier.tag(tag: String?): Modifier = testTag(NoteListTestTags.PREFIX, tag)
```

Production call sites read `Modifier.tag(NoteListTestTags.noteItem(note.id))`, resolving to `"note_list_note_item_<id>"` at runtime. The screen prefix avoids collisions between screens that are briefly composed together (nav transitions, dialogs, backstack); the id suffix avoids collisions between rows of the same list. Test code builds the identical string with the plain `testTag(NoteListTestTags.PREFIX, NoteListTestTags.noteItem(id))` function — production and tests share one source of truth, so neither hardcodes the joined string:

```kotlin
@Composable
fun NoteListScreen(
    state: NoteListState,
    onAction: (NoteListAction) -> Unit
) {
    when {
        state.isLoading -> LoadingIndicator(
            modifier = Modifier.tag(NoteListTestTags.LOADING_INDICATOR)
        )
        state.notes.isEmpty() -> EmptyState(
            modifier = Modifier.tag(NoteListTestTags.EMPTY_STATE)
        )
        else -> LazyColumn {
            items(items = state.notes, key = { it.id }) { note ->
                NoteItem(
                    note = note,
                    onClick = { onAction(OnNoteClick(note.id)) },
                    modifier = Modifier.tag(NoteListTestTags.noteItem(note.id))
                )
            }
        }
    }
}

// In a test
composeTestRule
    .onNodeWithTag(testTag(NoteListTestTags.PREFIX, NoteListTestTags.noteItem("1")))
    .performClick()
```

### Naming Convention

`snake_case` throughout. The object-level `PREFIX` is the screen name; individual constants/functions are just `<element>[_<role>]`.

### What to Tag

- Every interactable element — `Button`, `IconButton`, `TextField`, clickable rows — even ones with a stable, non-localized text label. Don't rely on text-based lookup (`onNodeWithText`) alone; text can be localized or change, a tag doesn't.
- Elements a test can't otherwise find: loading indicators, empty/error states, list item rows.
- Skip plain layout containers (root `Box`/`Column`, spacing wrappers) unless the container itself is interactable — don't tag scaffolding just for the sake of it.

---

## TextField

Text input state lives in the ViewModel. Every keystroke dispatches an Action:

```kotlin
// In the Screen composable
TextField(
    value = state.title,
    onValueChange = { onAction(NoteEditorAction.OnTitleChange(it)) }
)
```

The ViewModel updates state (and optionally persists to `SavedStateHandle`) in response to the Action — see the **android-presentation-mvi** skill for the full pattern.
