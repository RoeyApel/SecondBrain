# React Coding Standards

## Core Principles

- Treat components and hooks as pure functions of their props/state/context: same inputs must produce the same output.
- Never perform side effects during render (no mutating external variables, no DOM writes, no network calls in the component body).
- Never mutate props, state, or any value after it has been passed to a Hook or used in JSX — treat all of these as read-only.
- Only call components via JSX (`<Component />`), never invoke them as plain functions.
- Only call Hooks from the top level of a React function component or custom Hook — never inside conditions, loops, or nested functions, and never from plain JS functions.

## Architecture & Structure

- Decompose UI by drawing component boundaries around single-responsibility pieces; match component structure to the shape of the data model where possible.
- Build a static, non-interactive version first (props only, no state), then layer in state and interactivity.
- Data flows down via props; events flow up via callback props. Do not fight this one-way flow with ad-hoc global mutation.
- Determine state placement by finding the closest common parent of every component that needs it, and put the state there (or in a component created above it) — avoid state living higher than necessary.
- Prefer composition (children/render props) over deep prop drilling; introduce Context only for genuinely cross-cutting data (theme, auth, locale), not as a general prop-passing shortcut.

## Code Quality & Style

- Only keep something in state if it changes over time AND cannot be derived from existing props/state; compute everything else during render.
- Keep components small and named for what they render; extract a subcomponent when a component mixes unrelated concerns.
- Use a stable, unique `key` (not array index, unless the list is static and never reordered/filtered) when rendering lists.
- Prefer controlled inputs for form state you need to read or validate; use uncontrolled inputs (refs) only for simple, isolated cases.

## Error Handling

- Wrap sections of the UI that can fail independently in error boundaries so one failure doesn't blank the whole app.
- Handle expected failures (form validation, failed requests) via normal state and conditional rendering, not exceptions caught ad hoc in render.
- In event handlers, catch and surface errors from async calls explicitly (e.g., set an error state) rather than letting promise rejections go unhandled.

## Security

- Never pass unsanitized user input to `dangerouslySetInnerHTML`; avoid it entirely unless the HTML is trusted/sanitized server-side.
- Treat all externally supplied URLs (e.g., `href`, `src`) as untrusted; validate/allowlist schemes to avoid `javascript:` injection.
- Do not embed secrets or API keys in client-side code/bundles — anything shipped to the browser is public.

## Performance

- Do not reach for `useMemo`/`useCallback`/`React.memo` by default; only add them when a calculation is measurably slow with stable dependencies, or when passing a stable reference into a `memo`-wrapped child or as another Hook's dependency.
- Prefer computing derived values inline during render over caching them in state.
- Avoid creating new object/array/function literals as props to memoized children without memoizing them — this defeats the memoization.
- Move values used only inside an Effect into that Effect instead of memoizing them in the component body.

## Testing

- Test component behavior from the user's perspective (rendered output, interactions) rather than internal implementation details (state, private methods).
- Assert on accessible queries (role, label, text) rather than test-only selectors where practical.
- Cover error and loading states, not just the happy path, for any component that fetches or mutates data.

## Anti-Patterns to Avoid

- Using `useEffect` to compute derived state from props/state — calculate it during render or use `useMemo` instead.
- Using `useEffect` to respond to a specific user action (e.g., a click) — put that logic in the event handler.
- Using `useEffect` + `setState` to reset state when a prop changes — pass a different `key` to reset the subtree instead.
- Chaining multiple `useEffect`s that each set state to trigger the next — compute all resulting state in a single event handler.
- Calling Hooks conditionally or inside loops.
- Mutating state or props directly instead of creating new values.
- Using array index as `key` for lists that can reorder, insert, or delete items.

## Sources

- [Rules of React](https://react.dev/reference/rules)
- [Thinking in React](https://react.dev/learn/thinking-in-react)
- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [Managing State](https://react.dev/learn/managing-state)
- [useMemo reference](https://react.dev/reference/react/useMemo)
