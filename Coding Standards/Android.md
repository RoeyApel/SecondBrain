# Android Coding Standards

## Core Principles

- Follow the official app architecture: UI layer → Domain layer (optional) → Data layer.
- Follow unidirectional data flow (UDF): state flows down, events flow up to a single source of truth (SSOT).
- Assign exactly one owner (SSOT) per data type; only that owner mutates the data; expose it as immutable types elsewhere.
- Design each method, class, file, package, and module with a single, clearly defined responsibility and boundary.

## Architecture & Structure

- Never store application data or UI state in `Activity`, `Service`, or `BroadcastReceiver` — they are ephemeral and destroyed frequently.
- Use `ViewModel` as the state holder for UI; scope its lifetime to the UI element it serves.
- Do not put business logic in `Activity`/`Fragment`; keep it in ViewModels or the domain layer.
- Do not use Android `Context` directly inside ViewModels or domain classes — inject only what's needed, behind an interface where practical.
- Build the data layer from repositories, one per data type (e.g. `MoviesRepository`), each composed of one or more data sources (network, disk, database).
- Give each data source responsibility for exactly one source of truth (a single file, API, or database).
- Use the domain layer (use case/interactor classes) only for business logic that is complex or reused by multiple ViewModels — do not add it as a default layer for trivial logic.
- Use dependency injection (Hilt) rather than service locators or manual singletons; DI gives compile-time verification and framework-integrated dependency containers.
- Drive UI from persistent, lifecycle-independent data models, not from transient UI state, to support process death, configuration changes, and offline use.
- Use Jetpack libraries for common infrastructure (background work, DI, persistence) instead of reinventing boilerplate.

## Code Quality & Style

- Design for testability from the start: depend on interfaces, not concrete framework classes, so fakes/mocks can be substituted.
- Keep data-loading logic centralized in the repository responsible for that data type; do not scatter it across unrelated classes or packages.
- Do not expose a repository's or data source's internal implementation details (e.g. raw DTOs, database entities) past the layer boundary that owns them.
- Make each type explicitly responsible for its own concurrency policy (e.g. which dispatcher it runs on) rather than assuming callers manage it.

## Error Handling

- Surface data-layer failures (network, disk, database) as explicit result/error types through the repository, not as uncaught exceptions crossing layer boundaries.
- Validate all data coming from external storage, IPC, intents, and network responses — treat it as untrusted input regardless of source.
- Do not perform sensitive actions (auth, payments, state mutation) based solely on data from implicit intents or SMS; verify the sender/source first.

## Security

- Use internal storage for sensitive data; it is private to your app by default. Never use `MODE_WORLD_WRITEABLE`/`MODE_WORLD_READABLE`.
- Store only non-sensitive data on external storage, and treat anything read from it as untrusted input.
- Explicitly set `android:exported=false` on components (activities, services, receivers, providers) not intended for use by other apps; don't rely on defaults.
- Use parameterized queries (`query`/`update`/`delete` with selection args) for all database and `ContentProvider` access — never build SQL by string concatenation.
- Use explicit `Intent`s for starting services and for any security-sensitive IPC; implicit intents to services are prohibited on modern Android and are a security hazard generally.
- Use HTTPS (`HttpsURLConnection`) for all network traffic carrying sensitive data; never fall back to plaintext HTTP.
- Do not implement custom SSL/TLS or crypto primitives; use platform implementations (`HttpsURLConnection`, `SSLSocket` with hostname verification, Android Keystore, `SecureRandom`). Use certificate pinning for high-value connections.
- Never store passwords or long-lived user credentials on-device; use short-lived tokens, refresh them, and store any required secrets in Android Keystore.
- Do not log personally identifiable information (PII) or secrets; disable verbose logging in production builds.
- Never commit API keys or secrets to source control; load them via secure mechanisms (e.g. secrets-gradle-plugin, environment-specific build config) and store at rest via Keystore.
- Disable WebView JavaScript (`setJavaScriptEnabled`) unless explicitly required, and never bridge `addJavaScriptInterface` to untrusted or remote web content.
- Avoid dynamic code loading from outside the APK (network, external storage); loaded code inherits the app's full permissions.
- Use the Play Integrity API to detect tampered app builds or untrustworthy environments where app integrity matters (e.g. anti-fraud).

## Performance

- Preserve UI state across configuration changes (rotation, multi-window, foldables) instead of reloading data from scratch.
- Implement adaptive/canonical layouts and use window-size-class APIs rather than hardcoding layouts per device.
- Use `WorkManager`/`JobScheduler` for deferrable or guaranteed background work instead of long-running unbound services.
- Avoid holding long-lived references to `Activity`/`View`/`Context` from singletons, static fields, or long-lived coroutines/observers — this leaks memory.

## Testing

- Write small (unit), medium (integration), and big (end-to-end/UI) tests; keep most tests small and fast, per the test-size pyramid.
- Write local (JVM/Robolectric) unit tests for ViewModels, use cases, and repositories by injecting fakes for their dependencies — do not require a device/emulator for pure logic.
- Write instrumented tests only for behavior that genuinely needs the Android framework or device (UI interaction, real SQLite, sensors).
- Enable testability by depending on interfaces and injecting dependencies, so fakes/mocks can replace real implementations in tests.
- Use Espresso or the Compose testing APIs (`composeTestRule`) for instrumented UI assertions, not manual/ad-hoc verification.

## Anti-Patterns to Avoid

- God Activities/Fragments that own networking, persistence, and UI logic directly.
- Storing mutable app state in a singleton outside the designated SSOT/repository.
- Using a service locator instead of constructor/field injection.
- Building SQL or `ContentProvider` selections via string concatenation with user input.
- Broad, always-on permission requests instead of the minimum required, requested at the point of need.
- Relying on SMS for data transfer or as an authentication factor.
- Catching and swallowing exceptions at the data layer instead of propagating a typed result/error to the caller.

## Sources

- [Guide to app architecture](https://developer.android.com/topic/architecture)
- [Security tips](https://developer.android.com/privacy-and-security/security-tips)
- [Testing fundamentals](https://developer.android.com/training/testing/fundamentals)
