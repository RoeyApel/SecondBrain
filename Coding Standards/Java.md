# Java Coding Standards

## Core Principles

- Prefer immutability by default: `final` fields, immutable collections (`List.of`, `Map.of`), and records for data carriers.
- Favor composition over inheritance; use interfaces to define behavior contracts.
- Keep classes focused on a single responsibility; keep methods short and testable.
- Use modern language features (records, sealed types, pattern matching, `var`, text blocks) where they reduce boilerplate or improve safety — don't use them gratuitously where a plain class or explicit type is clearer.

## Architecture & Structure

- Use `record` for immutable data aggregates (DTOs, value objects) instead of hand-written classes with boilerplate accessors, `equals`, `hashCode`, `toString`. Add a compact constructor for validation.
- Use `sealed` interfaces/classes to model closed, exhaustive type hierarchies (e.g., result/state types); permitted subtypes must be `final`, `sealed`, or `non-sealed`.
- Use pattern matching for `switch`/`instanceof` to eliminate manual casting; rely on sealed-type exhaustiveness instead of a `default` branch when the hierarchy is closed.
- Depend on abstractions (interfaces) at module/package boundaries; wire concrete implementations via dependency injection rather than static singletons or `new` scattered through business logic.
- Organize packages by feature/domain, not by technical layer alone (avoid a single flat `utils` or `service` dumping ground).

## Code Quality & Style

- Use `var` only when the inferred type is obvious from the right-hand side; do not use it when it obscures the type at a glance.
- Follow standard Java naming conventions (`camelCase` for methods/fields, `PascalCase` for types, `UPPER_SNAKE_CASE` for constants).
- Prefer text blocks (`"""`) over concatenated strings for multi-line literals (SQL, JSON, HTML fragments).
- Minimize public API surface: default to package-private/private; expose only what callers need.
- Avoid returning `null` from methods that can reasonably return an empty collection or `Optional` instead.

## Error Handling

- Use unchecked exceptions for programming errors/invariant violations; reserve checked exceptions for recoverable conditions the caller is expected to handle.
- Never silently swallow exceptions (empty `catch` blocks); at minimum log with context, or rethrow/wrap with additional information.
- Use try-with-resources for any `AutoCloseable`/`Closeable` resource; never manage streams, connections, or locks with manual `finally` cleanup.
- Use `Optional` as a return type to signal "may be absent" for values, not as a field type, method parameter, or for collections (return an empty collection instead).
- Include actionable context in exception messages; do not expose internal/stack details in messages surfaced to end users.

## Security

- Use an allowlist approach for input validation combined with output encoding/escaping; never rely on blacklisting.
- Always use parameterized queries / prepared statements (JDBC `PreparedStatement`, JPA named parameters) — never build SQL via string concatenation.
- Never deserialize untrusted data with raw `ObjectInputStream`; if unavoidable, override `resolveClass()` to allowlist expected types, and mark non-deserializable domain classes with a `readObject()` that throws.
- Mark sensitive fields (`private transient`) to exclude them from default Java serialization.
- Never write custom cryptography; use vetted libraries (JCA/JCE providers, Google Tink) with AES-GCM and unique nonces per operation.
- Never hardcode credentials or keys; use a secret manager (cloud provider KMS/secrets manager or equivalent).
- Use parameterized/structured logging (`logger.warn("Login failed for {}", user)`), not string concatenation, to prevent log injection; avoid logging sensitive data.
- Keep dependencies current and monitor advisories (e.g., via OWASP Dependency-Check or equivalent); do not pin to known-vulnerable versions.

## Performance

- Prefer virtual threads (`Executors.newVirtualThreadPerTaskExecutor()`) for high-concurrency I/O-bound workloads instead of manually sized platform-thread pools.
- Choose between `synchronized` and `java.util.concurrent.locks` based on the actual need (fairness, read-write locks, interruptible acquisition) — on current JDKs, `synchronized` no longer forces virtual-thread pinning as of JDK 24, so pinning avoidance alone is not a reason to prefer explicit locks.
- Narrow lock scope to the minimum critical section; avoid performing I/O or other blocking calls while holding a lock.
- Avoid premature optimization; profile before micro-optimizing hot paths, and prefer clear code unless a measured bottleneck justifies complexity.
- Prefer streaming/lazy evaluation (`Stream`) for large or pipeline-style data processing, but avoid `Stream` for simple loops where a for-each is clearer and faster.

## Testing

- Write unit tests for business logic in isolation; use test doubles (mocks/fakes) only at true architectural boundaries (external services, I/O), not to isolate every collaborator.
- Prefer constructor-based dependency injection so classes are trivially testable without a DI framework in unit tests.
- Cover exception paths and edge cases explicitly, not just the happy path.
- Keep tests deterministic and independent of execution order or shared mutable state.

## Anti-Patterns to Avoid

- God classes / utility classes that accumulate unrelated static methods.
- Catching `Exception` or `Throwable` broadly instead of specific exception types.
- Using exceptions for normal control flow.
- Public mutable static fields (shared mutable global state).
- Overusing `Optional` (as a field, parameter, or in collections) instead of as a method return type.
- Manual resource management instead of try-with-resources.
- Reflection-based hacks to bypass encapsulation instead of fixing the API.
- Treating `Effective Java`-era idioms (e.g., manual builder boilerplate for simple immutable data, hand-written `equals`/`hashCode`) as the default when a `record` now expresses the same intent with less code.

## Sources

- [Oracle Java Language Updates — Records](https://docs.oracle.com/en/java/javase/21/language/records.html)
- [JEP 409: Sealed Classes](https://openjdk.org/jeps/409)
- [JEP 441: Pattern Matching for switch](https://openjdk.org/jeps/441)
- [Oracle — Pattern Matching for switch](https://docs.oracle.com/en/java/javase/21/language/pattern-matching-switch.html)
- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [OWASP Java Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Java_Security_Cheat_Sheet.html)
- [OWASP Deserialization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html)

Supplementary (not authoritative for modern-Java features, used only for well-established general idioms like immutability discipline and API design restraint): Joshua Bloch, *Effective Java*, 3rd Edition.
