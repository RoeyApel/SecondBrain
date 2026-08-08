# TypeScript Coding Standards

## Core Principles

- Treat TypeScript's type system as a design tool, not an afterthought bolted onto JavaScript.
- Prefer compile-time guarantees over runtime checks wherever the type system can express the constraint.
- Let the compiler catch mistakes: enable strict checking and treat type errors as build failures, not warnings to suppress.

## Architecture & Structure

- Model domain state as discriminated unions (a shared literal `kind`/`type` field) instead of one interface with many optional properties. This enables exhaustive narrowing and removes whole classes of "impossible state" bugs.
- Use ES modules (`import`/`export`) exclusively; do not use TypeScript `namespace`/`module` syntax in new code.
- Keep exported function signatures fully annotated (parameter and return types) — this is the module's contract and should not rely on inference.
- Use `as const` on literal object/array values you want treated as fixed literal types rather than widened primitives.

## Code Quality & Style

- Default to `interface` for object shapes; switch to `type` when you need unions, intersections, tuples, or mapped/conditional types. Rationale (TS handbook): "use `interface` until you need to use features from `type`."
- Prefer `unknown` over `any` for values of uncertain type; narrow with `typeof`, `instanceof`, `in`, or user-defined type predicates (`x is T`) before use.
- Prefer type inference for local variables with obvious initializers; add explicit annotations at function boundaries and for anything non-obvious.
- Use `readonly` on properties, arrays (`readonly T[]`), and tuples that must not be mutated after construction.
- Prefer literal union types (`"left" | "right" | "center"`) over `enum` for closed sets of string constants — they erase at compile time and interoperate more cleanly with plain JS/JSON.
- Use generics to preserve type relationships between inputs and outputs; do not add a generic parameter that is only used once (it adds noise without adding safety).
- Use utility types (`Partial`, `Pick`, `Omit`, `Record`, `ReturnType`, etc.) to derive related types instead of hand-duplicating shapes.

## Error Handling

- Type `catch` variables as `unknown` (the modern default under `useUnknownInCatchVariables`) and narrow before accessing properties — never assume the caught value is an `Error`.
- Do not use the non-null assertion operator (`!`) to silence a possibly-`null`/`undefined` value unless you can justify why the compiler cannot see an invariant that truly holds; prefer a real guard or a narrowed check.
- Avoid type assertions (`as T`) to force a value into a shape you haven't actually validated — this is a compile-time-only cast with no runtime check, and a mismatch will fail elsewhere with a confusing error.
- Return or throw explicit error types/discriminated `Result`-like unions at API boundaries instead of letting failures propagate as untyped `any`.

## Security

- Never widen a type to `any` solely to work around a compiler error at a trust boundary (parsing user input, deserializing JSON, reading env vars) — validate the shape at runtime (e.g., with a schema validator) and let the validator produce the type.
- Do not use `as` to assert the type of unvalidated external input (network responses, `JSON.parse` results); an assertion does not check anything at runtime and can mask malformed or malicious payloads as trusted data.
- Keep secrets and environment-derived config out of committed type-safe "constants" files — type safety does not protect against secrets ending up in source control.

## Performance

- Prefer `interface extends` over intersection types (`&`) for object composition; the compiler can cache and reuse interface relationships more efficiently, which matters in large codebases' type-checking time.
- Avoid deeply recursive or combinatorially large conditional/mapped types — they slow down `tsc` and editor responsiveness disproportionately to their value.
- Scope `tsconfig.json` `include`/`exclude` and use project references for large codebases so incremental builds only re-check what changed.

## Testing

- Write tests in TypeScript so test code is type-checked against the same interfaces as production code — a signature change that breaks a test should be a type error, not just a runtime failure.
- Do not use `any` or blanket `@ts-ignore` in test files to bypass type errors when mocking; construct properly typed fixtures or use the utility types (`Partial<T>`, `Pick<T,...>`) to build partial mocks explicitly.
- Enable `noUnusedLocals`/`noUnusedParameters` so dead test setup code and stale mocks are caught automatically.

## Anti-Patterns to Avoid

- Do not disable `strict` mode (or leave it unset) in new projects; start every project with `"strict": true`.
- Do not use `any` as a quick fix for a type error — it disables checking for that value and everything derived from it.
- Do not model variant/optional-heavy state with one big interface full of `?` properties when a discriminated union expresses the same data more safely.
- Do not overuse non-null assertions (`!`) or `as` casts to make the compiler stop complaining — treat repeated need for either as a signal the types are modeled incorrectly.
- Do not use `enum` by default; prefer literal union types unless you specifically need enum-specific runtime behavior.
- Do not leave function parameters or exported APIs untyped and rely on inference alone — inference is for internal/local code, not public contracts.

## Sources

- [TypeScript Handbook — Everyday Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html)
- [TypeScript Handbook — Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- [TSConfig Reference](https://www.typescriptlang.org/tsconfig)
