# General Coding Standards

## Core Principles

- Favor changes that clearly improve overall code health over changes that chase perfection.
- Base design and style decisions on engineering principles and project conventions, not personal preference.
- When no explicit rule applies, match existing patterns in the codebase over introducing a new one.
- Keep functions, classes, and modules focused on a single responsibility.
- Prefer simple, obvious solutions over clever ones; optimize for the next reader.

## Architecture & Structure

- Keep changes small and self-contained — one logical change per commit/PR (~100 lines is reasonable, ~1000 is too large).
- Design module boundaries around clear ownership of data and behavior; avoid circular dependencies.
- Isolate side effects (I/O, network, global state) from pure logic to keep the core testable.
- Introduce abstractions only after a real duplication or extension need exists, not speculatively.

## Code Quality & Style

- Use descriptive, unambiguous names for variables, functions, and types; avoid abbreviations that aren't domain-standard.
- Write commit/PR descriptions that explain *why*, not just *what*.
- Delete dead code and unused dependencies instead of commenting them out.
- Keep comments limited to non-obvious rationale (constraints, invariants, workarounds); do not narrate what the code already says.
- Run formatters/linters and fix warnings rather than suppressing them, unless the suppression is justified inline.

## Error Handling

- Fail fast and loudly on programmer errors (invalid state, contract violations); handle recoverable errors explicitly.
- Never silently swallow exceptions/errors — log or propagate them.
- Preserve original error context (cause/stack trace) when wrapping or rethrowing.
- Handle errors at the layer that has enough context to decide the correct response; avoid handling the same error redundantly at multiple layers.

## Security

- Treat all external input (user input, network responses, files, env vars) as untrusted; validate on the trusted/server side, using allow-lists over deny-lists.
- Never hardcode secrets, credentials, or connection strings in source code; load them from a secrets manager or environment configuration.
- Enforce authentication and authorization on every request/operation via a single, centralized mechanism — never rely on client-side checks alone.
- Use parameterized queries/APIs for any interpreted context (SQL, shell, HTML, LDAP); never build these strings by concatenating untrusted input.
- Do not pass user-supplied data to dynamic code-execution functions (`eval`, dynamic `exec`, reflection-based invocation) under any circumstance.
- Apply least privilege to services, database users, and API tokens; scope credentials as narrowly as possible.
- Do not leak sensitive information (stack traces, internal paths, secrets) in error messages or logs returned to users.
- Log security-relevant events (auth attempts, access-control failures, input-validation failures) without logging sensitive data itself (passwords, tokens, PII).

## Performance

- Measure before optimizing; do not hand-optimize code without a profiler or benchmark showing a real bottleneck.
- Choose algorithms and data structures appropriate to the expected data volume up front — this is a design decision, not a later optimization.
- Avoid unnecessary work in hot paths: redundant I/O, repeated parsing/serialization, N+1 calls.
- Prefer streaming/paginating large datasets over loading them entirely into memory.

## Testing

- Cover new behavior with automated tests as part of the same change, not as follow-up work.
- Write tests that assert observable behavior/contracts, not implementation details, so refactors don't break them unnecessarily.
- Include tests for edge cases and failure paths, not only the happy path.
- Keep tests deterministic and independent of execution order, external network access, or wall-clock time.

## Anti-Patterns to Avoid

- Large, multi-purpose changes that mix refactoring with behavioral changes.
- Deny-list-based input validation.
- Broad `catch`/`except` blocks that discard the original error.
- Committing secrets, API keys, or credentials to version control.
- Building queries or commands via string concatenation of untrusted input.
- Trusting client-side validation or client-side access control as a security boundary.
- Premature abstraction or configuration options for requirements that don't yet exist.
- Copy-pasting logic instead of factoring out a shared, well-named unit once duplication is real.

## Sources

- [Google Engineering Practices — Code Review Standard](https://google.github.io/eng-practices/review/reviewer/standard.html)
- [Google Engineering Practices — Small CLs](https://google.github.io/eng-practices/review/developer/small-cls.html)
- [Google Engineering Practices — Writing Good CL Descriptions](https://google.github.io/eng-practices/review/developer/cl-descriptions.html)
- [OWASP Secure Coding Practices — Quick Reference Guide](https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/stable-en/)
- [OWASP Top 10:2025 — Introduction](https://owasp.org/Top10/2025/0x00_2025-Introduction/)
