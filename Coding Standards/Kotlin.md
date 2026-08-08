# Kotlin Coding Standards

## Core Principles

- Prefer `val` over `var`; treat mutability as an explicit, justified choice.
- Prefer immutable collection types (`List`, `Set`, `Map`) over mutable ones (`ArrayList`, `HashSet`) unless mutation is required.
- Prefer expression-style code (`if`/`when` as expressions, single-expression functions) over statement-style with explicit `return`.
- Always specify visibility modifiers explicitly on public API members; don't rely on the default `public`.
- Always specify explicit return types and property types on public/exposed declarations.

## Architecture & Structure

- Name a file after the single class/interface it contains; use a descriptive PascalCase name (not `Utils`/`Util`) for files holding multiple top-level declarations.
- Order class members: properties and init blocks, then secondary constructors, then methods, then companion object, then nested classes.
- Use `sealed class`/`sealed interface` to model closed sets of states or results (e.g. UI state, network result) instead of open class hierarchies or boolean/enum flags.
- Use `data class` for value-holding types that need `equals`/`hashCode`/`copy`; do not add data-class semantics to types with identity or mutable internal state.
- Use extension functions to add behavior to types you don't own, and restrict their visibility (`private`/`internal`) unless they're intended as public API.
- Avoid giving a factory function the same name as the class unless it has no special construction semantics; otherwise name it descriptively (e.g. `Point.fromPolar(...)`).
- Use `typealias` to give meaningful names to function types and complex generics.

## Code Quality & Style

- Follow official naming conventions: `UpperCamelCase` for classes/objects, `lowerCamelCase` for functions/properties/variables, `UPPER_SNAKE_CASE` (or `SCREAMING_CASE`) only for true `const val` constants.
- Prefer `when` over chained `if`/`else if` once there are 3+ branches; keep `if` for binary conditions.
- Use string templates (`"$name has ${children.size}"`) instead of string concatenation.
- Use multiline strings with `trimIndent()`/`trimMargin()` instead of embedding `\n` escapes.
- Prefer named arguments for calls with multiple parameters of the same type or multiple boolean parameters.
- Prefer default parameter values over overloaded function variants.
- Use `..<` for exclusive upper bounds (`0..<n`) instead of `0..n - 1`.
- Prefer higher-order functions (`filter`, `map`, `fold`, etc.) over manual loops, except when a plain `for` loop is clearer (e.g. side-effecting iteration, nullable receiver, or needing early `break`/`continue`).
- Prefer `Sequence` over eager collection operations when chaining multiple transformations on large collections, to avoid intermediate list allocations.
- Choose scope functions deliberately, not habitually:
  - `let` — null-safe chaining (`x?.let { ... }`) or introducing a local name for an expression result.
  - `apply` — configuring/initializing an object, returns the receiver.
  - `also` — side effects (logging, validation) while chaining, returns the receiver.
  - `run` — computing a result from a receiver's members.
  - `with` — grouping multiple calls on a non-null object when no chaining/return is needed.
  - Do not nest scope functions or chain several together; it obscures which `this`/`it` is in scope.
- Provide KDoc for all public types and public/protected members.

## Error Handling

- Avoid `!!`; it converts a recoverable null case into an unhandled `NullPointerException`. Use safe calls (`?.`), `?:` with a fallback, or an early return/guard clause instead.
- Model expected failure as return values (`sealed class Result`, nullable types) rather than exceptions when failure is a normal, anticipated outcome; reserve exceptions for truly exceptional/programmer-error conditions.
- Never catch `CancellationException` and swallow it inside a coroutine — rethrow it, since it drives structured cancellation.
- For `async`, the exception is stored in the `Deferred` and only surfaces on `await()`; do not fire-and-forget an `async` you never await.
- Install `CoroutineExceptionHandler` only on root coroutines (top-level `launch`/scope), not on children — child exceptions propagate to the parent and handlers on children are ignored.
- Use `supervisorScope`/`SupervisorJob` when sibling coroutines must fail independently (e.g. independent UI widgets or independent tasks); otherwise a child's failure cancels the whole scope by default.

## Security

- Never store secrets, API keys, or credentials in source; read them from environment/secret-management systems.
- Validate and sanitize any input crossing a trust boundary (deserialized JSON, user input, external service responses) before using it in logic or persistence.
- Avoid logging sensitive data (tokens, PII) even in debug builds.

## Performance

- Avoid unnecessary object allocation in hot paths (e.g. creating lambdas/collections inside loops that don't need to be recreated).
- Use `Sequence` for large collection pipelines with multiple chained operations; use direct collection operations for small collections where eager evaluation is simpler and fine.
- Avoid blocking calls inside coroutines; use suspending equivalents (e.g. non-blocking IO) so the underlying thread isn't held.
- Avoid `GlobalScope` for launching coroutines — it decouples lifetime from any structured scope and risks leaks; launch from a scope tied to the relevant lifecycle instead.

## Testing

- Name test functions descriptively; backtick-quoted names (`` `ensure everything works`() ``) are acceptable and encouraged for readability over verbose camelCase.
- Test coroutine code with structured-concurrency-aware test utilities (e.g. `runTest`) rather than blocking sleeps or ad hoc thread coordination.
- Prefer testing sealed-class/result states exhaustively (`when` without an `else` branch) so new states cause compile-time test failures.

## Anti-Patterns to Avoid

- Don't use `!!` as a substitute for proper null handling.
- Don't use `GlobalScope.launch`/`async` for application logic.
- Don't put a `CoroutineExceptionHandler` on a child coroutine expecting it to intercept the exception before it reaches the parent.
- Don't use mutable collection types or `var` as the default choice — justify mutability, don't default to it.
- Don't overload functions to simulate optional parameters; use default parameter values.
- Don't nest or chain multiple scope functions (`x.let { it.also { ... }.run { ... } }`) — refactor into explicit statements.
- Don't give files/classes meaningless names like `Utils.kt` or `Helper.kt`.

## Sources

- [Kotlin Coding Conventions](https://kotlinlang.org/docs/coding-conventions.html)
- [Kotlin Scope Functions](https://kotlinlang.org/docs/scope-functions.html)
- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [Kotlin Exception Handling in Coroutines](https://kotlinlang.org/docs/exception-handling.html)
- [Android Kotlin Style Guide](https://developer.android.com/kotlin/style-guide)
