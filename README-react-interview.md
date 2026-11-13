# React Interview Questions & Answers (Senior / Architect Level)

> Covers fundamentals, hooks, performance, state management, architecture, testing, security, and system design. Each answer is written the way you would say it in an interview: short, precise, with a trade-off where one matters.

## Table of Contents
1. [Core Fundamentals](#1-core-fundamentals)
2. [Hooks](#2-hooks)
3. [Rendering & Performance](#3-rendering--performance)
4. [State Management](#4-state-management)
5. [Data Fetching & Async](#5-data-fetching--async)
6. [React 18 / 19 Features](#6-react-18--19-features)
7. [Architecture & Design Patterns](#7-architecture--design-patterns)
8. [Routing, SSR & Next.js](#8-routing-ssr--nextjs)
9. [Testing](#9-testing)
10. [Security](#10-security)
11. [TypeScript with React](#11-typescript-with-react)
12. [Tricky / Output-Based Questions](#12-tricky--output-based-questions)
13. [System Design Questions](#13-system-design-questions)
14. [Behavioral Questions for Architects](#14-behavioral-questions-for-architects)
15. [Quick Revision Cheat Sheet](#15-quick-revision-cheat-sheet)

---

## 1. Core Fundamentals

**Q1. What is React, and why use it?**
A declarative, component-based UI library. You describe what the UI should look like for a given state, and React works out how to update the DOM. Benefits: composability, a predictable one-way data flow, and a large ecosystem. It is a library, not a framework: routing, data fetching, and structure are your choices (or a framework's, such as Next.js).

**Q2. What is the Virtual DOM, and how does reconciliation work?**
The Virtual DOM is a lightweight in-memory tree of React elements. On a state change React builds a new tree, **diffs** it against the previous one, and applies only the minimal changes to the real DOM. Two heuristics keep diffing O(n): elements of different types produce different trees, and `key` identifies elements across renders in a list.

**Q3. Why are keys important? Why not use array index?**
Keys let React match elements between renders. With index keys, inserting, removing, or reordering items makes React reuse the wrong component instances, causing wrong state (e.g., input values stuck to the wrong row) and wasted renders. Use a stable, unique ID from the data.

**Q4. What is the difference between props and state?**
Props are read-only inputs from a parent. State is data owned and changed by the component itself (via `useState`/`useReducer`). A change in either triggers a re-render.

**Q5. Controlled vs uncontrolled components?**
Controlled: React state is the single source of truth for the input value (`value` + `onChange`). Uncontrolled: the DOM holds the value, and you read it via a `ref` or on submit. Controlled suits validation and dependent fields. Uncontrolled suits simple forms, file inputs, and performance-sensitive large forms (or use a library such as React Hook Form, which is uncontrolled under the hood).

**Q6. What is JSX?**
Syntax sugar for `React.createElement` (or the automatic `jsx()` runtime). It is compiled by Babel/TypeScript/SWC. It is an expression, so it can be assigned, returned, and passed around.

**Q7. What is lifting state up?**
When two components need the same data, move the state to their closest common ancestor and pass it down via props. If that becomes painful (prop drilling), consider context or a state library.

**Q8. Class components vs function components?**
Function components with hooks are the standard. Classes are still supported, and **error boundaries still require a class** (unless you use a library such as `react-error-boundary`). Hooks give better logic reuse than HOCs and render props.

**Q9. What is the component lifecycle in terms of hooks?**
Mount, update, and unmount map to `useEffect` with the dependency array: `[]` runs after mount (and its cleanup on unmount), `[dep]` runs when `dep` changes. Cleanup runs before the next effect and on unmount.

**Q10. What are Fragments, and why use them?**
`<>...</>` lets a component return multiple siblings without an extra DOM node. Use `<Fragment key=...>` when you need a key.

---

## 2. Hooks

**Q11. Rules of Hooks, and why do they exist?**
Call hooks only at the top level (not in loops, conditions, or nested functions) and only from React functions or custom hooks. React tracks hooks by **call order** per component. Changing the order between renders corrupts that mapping.

**Q12. `useState`: what is the functional update form and when do you need it?**
`setCount(c => c + 1)`. Use it when the next state depends on the previous state, especially with multiple updates in one tick or stale closures. `setCount(count + 1)` three times in a row only increments once because `count` is the same snapshot.

**Q13. `useEffect` in depth: when does it run, and what are the common pitfalls?**
It runs **after paint**, after the render commits. Pitfalls:
- **Missing dependencies** leads to stale closures. Use the `exhaustive-deps` lint rule.
- **Unstable dependencies** (objects or functions recreated each render) cause infinite loops. Stabilize with `useMemo`/`useCallback`, or move them out of the component.
- **No cleanup** leaks subscriptions, timers, and listeners.
- **Using effects for derived state.** Compute during render instead.
- **Race conditions** in fetches. Use an abort flag or `AbortController`.

**Q14. `useEffect` vs `useLayoutEffect`?**
`useLayoutEffect` runs synchronously after DOM mutations and **before paint**. Use it to measure layout and mutate the DOM without flicker. `useEffect` runs after paint and is the default. `useLayoutEffect` blocks painting, so use it sparingly. It also does not run on the server.

**Q15. `useMemo` vs `useCallback` vs `React.memo`?**
- `useMemo(fn, deps)` memoizes a **computed value**.
- `useCallback(fn, deps)` memoizes a **function reference** (`useMemo(() => fn, deps)`).
- `React.memo(Component)` skips re-rendering if props are shallow-equal.
They are **optimizations, not semantic guarantees**. `useCallback` only helps when the function is passed to a memoized child or used as a dependency. Measure first. (React Compiler can automate much of this.)

**Q16. `useRef`: what is it for?**
A mutable container whose `.current` persists across renders **without triggering a re-render**. Uses: DOM access, storing timers or previous values, instance-like variables, and the latest-callback pattern.

**Q17. `useReducer` vs `useState`?**
`useReducer` fits complex or related state with multiple transitions, where next state depends on a named action. It centralizes logic, is easy to test (a pure function), and pairs well with context for a mini-store.

**Q18. `useContext`: what are the performance caveats?**
Every consumer re-renders when the context **value** changes (by reference). Mitigations: split contexts (state vs dispatch, or by domain), memoize the value, keep context for low-frequency data (theme, auth, locale), and use a selector-capable store (Zustand, Redux, `use-context-selector`) for high-frequency data.

**Q19. What is a custom hook, and when do you write one?**
A function starting with `use` that composes other hooks to share **stateful logic** (not state itself, as each call has its own state). Examples: `useDebounce`, `useLocalStorage`, `useFetch`, `useOnClickOutside`.

**Q20. `useId`, `useTransition`, `useDeferredValue`, `useSyncExternalStore`: what are they for?**
- `useId`: stable unique IDs that match between server and client (accessibility attributes).
- `useTransition`: mark a state update as non-urgent so the UI stays responsive.
- `useDeferredValue`: defer re-rendering of a slow part using a lagging value.
- `useSyncExternalStore`: subscribe safely to external stores without tearing in concurrent rendering. State libraries use it.

**Q21. `useImperativeHandle` and `forwardRef`?**
They expose a controlled imperative API from a child (e.g., `focus()`, `reset()`) to a parent. Use them rarely. Prefer props and state. (In React 19, `ref` can be passed as a regular prop to function components.)

---

## 3. Rendering & Performance

**Q22. When does a component re-render?**
When its state changes, its parent re-renders (even if props are identical, unless memoized), or a context it consumes changes. A re-render does not necessarily mean a DOM update.

**Q23. How do you diagnose and fix slow React apps?**
1. **Measure:** React DevTools Profiler, Chrome Performance, Lighthouse, and Web Vitals (LCP, INP, CLS).
2. **Find the cause:** unnecessary re-renders, expensive renders, huge lists, large bundles, or waterfalls.
3. **Fix, in order of payoff:**
   - Move state down or lift content up, so fewer components re-render (**composition before memoization**).
   - Virtualize long lists (`react-window`, `TanStack Virtual`).
   - Code-split with `React.lazy`/`Suspense` and route-based splitting.
   - Memoize where measured (`memo`, `useMemo`, `useCallback`).
   - Debounce/throttle inputs; use `useDeferredValue`/`useTransition`.
   - Optimize images, fonts, and third-party scripts; use server rendering or streaming.
   - Analyze the bundle and tree-shake dependencies.

**Q24. What is code splitting?**
Splitting the bundle into chunks loaded on demand: `const Page = lazy(() => import('./Page'))` wrapped in `<Suspense fallback=...>`. Split at route boundaries first, then at heavy components (charts, editors).

**Q25. How do you render a list of 100,000 items?**
Virtualization (windowing): render only visible rows plus a small overscan. Pair it with pagination or infinite scroll on the data side, stable keys, and memoized row components.

**Q26. What is "prop drilling," and how do you avoid it?**
Passing props through many layers that don't use them. Options: composition (`children`/slots), context, or a state library. Don't reach for global state first. Often restructuring components is enough.

**Q27. What are the pitfalls of `React.memo`?**
Shallow comparison fails if you pass new object, array, or function references each render. Memoization has its own cost, and benefits only appear where renders are expensive and props are stable. Inline `children` JSX also breaks it.

**Q28. What is the React Compiler?**
A build-time tool that automatically memoizes components and values, reducing the need for manual `useMemo`/`useCallback`/`memo`. It relies on following the Rules of React (pure render, no mutation of props or state).

---

## 4. State Management

**Q29. How do you decide where state should live?**
Categorize it:
- **Local UI state:** `useState` in the component.
- **Shared UI state:** lift it up or use context.
- **Server/cache state:** TanStack Query, SWR, or RTK Query. This is not "global state"; it is a cache with its own lifecycle.
- **URL state:** the router (filters, pagination, tabs) so views are shareable and bookmarkable.
- **Form state:** React Hook Form or Formik.
- **Global client state:** Zustand, Redux Toolkit, or Jotai, only when truly shared and complex.

**Q30. Redux vs Context vs Zustand: how do you choose?**
| | Context | Redux Toolkit | Zustand/Jotai |
|---|---|---|---|
| Best for | Low-frequency values (theme, auth) | Large apps, many teams, strict patterns, devtools and time travel | Lightweight global state, minimal boilerplate |
| Re-render control | Poor without splitting | Selectors | Selectors / atoms |
| Boilerplate | Low | Medium (reduced by RTK) | Very low |
| Middleware/ecosystem | None | Rich (RTK Query, sagas, listeners) | Smaller |

**Q31. Explain Redux's data flow.**
View dispatches an **action**, a **reducer** (pure function) computes the new state, the **store** updates, and subscribed components re-render via selectors. Redux Toolkit adds `createSlice` (Immer-based immutable updates), `configureStore`, and RTK Query.

**Q32. Why immutability?**
React and Redux detect changes by reference comparison. Mutating in place means the reference doesn't change, so no re-render, and it breaks memoization and time-travel debugging. Use spread, `map`/`filter`, or Immer.

**Q33. Server state vs client state: why use TanStack Query?**
It handles caching, deduplication, background refetch, stale-while-revalidate, retries, pagination, optimistic updates, and invalidation, so you don't hand-roll `useEffect` + `useState` + loading flags, and you avoid duplicating server data in Redux.

---

## 5. Data Fetching & Async

**Q34. How do you fetch data safely in an effect?**
```jsx
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal })
    .then(r => r.json())
    .then(setData)
    .catch(e => { if (e.name !== 'AbortError') setError(e); });
  return () => controller.abort();   // avoids races and setState after unmount
}, [url]);
```
In production, prefer a data library or framework loaders over hand-written effects.

**Q35. What is a request waterfall, and how do you avoid it?**
Sequential fetches where each starts only after the previous component renders (parent fetches, then child mounts, then child fetches). Avoid it by hoisting fetches to the route/loader level, fetching in parallel, prefetching, or using server components or streaming.

**Q36. What are optimistic updates?**
Update the UI immediately assuming success, then reconcile or roll back on failure. With TanStack Query: `onMutate` (snapshot and set), `onError` (rollback), and `onSettled` (invalidate).

**Q37. What are Suspense and error boundaries for?**
Suspense declaratively shows a fallback while something (lazy code, data via a Suspense-enabled library or framework) is loading. Error boundaries catch rendering errors in their subtree and show a fallback UI. Together they give loading and error handling at chosen boundaries instead of in every component. Error boundaries do **not** catch errors in event handlers, async code, or SSR.

---

## 6. React 18 / 19 Features

**Q38. What is concurrent rendering?**
React can prepare multiple versions of the UI, interrupt low-priority rendering for urgent updates (typing, clicks), and resume later. It enables `useTransition`, `useDeferredValue`, and streaming Suspense.

**Q39. What is automatic batching?**
In React 18, state updates are batched everywhere (event handlers, timeouts, promises, native handlers), resulting in a single re-render. Previously this was only inside React event handlers. Use `flushSync` to opt out.

**Q40. What are React Server Components (RSC)?**
Components that run only on the server (at build or request time), can access databases and secrets directly, and send serialized output to the client, so **their code is not shipped in the bundle**. Client components (`'use client'`) handle interactivity. Benefits: smaller bundles and direct data access. Constraints: no state, effects, or browser APIs in server components. They are used through frameworks (Next.js App Router).

**Q41. What are Server Actions, and what do `useActionState`, `useFormStatus`, and `useOptimistic` do (React 19)?**
Server Actions are functions that run on the server, invoked from forms or client code. `useActionState` manages form action state and results, `useFormStatus` reads the pending state of the parent form, and `useOptimistic` shows optimistic UI during a pending action. React 19 also adds the `use()` API (read promises and context during render) and ref as a prop.

**Q42. Strict Mode double-invoking effects: why?**
In development, React mounts, unmounts, and remounts components to surface impure renders and missing effect cleanup. It's a dev-only check. If it breaks your code, the cleanup logic is wrong.

---

## 7. Architecture & Design Patterns

**Q43. Composition vs inheritance?**
React favors composition: `children`, slot props, and render props. Components accept other components or elements, avoiding rigid hierarchies.

**Q44. HOC vs render props vs custom hooks?**
All reuse logic. HOCs wrap components (risk: wrapper hell, prop collisions). Render props pass a function child (risk: nesting). **Custom hooks** are the modern default: flat, composable, and typed easily. HOCs still appear in libraries (e.g., `connect`, `withRouter`).

**Q45. Compound components pattern?**
Related components share implicit state via context, giving a flexible API: `<Tabs><Tabs.List/><Tabs.Panel/></Tabs>`. It is used in design systems.

**Q46. Container/presentational (smart/dumb) pattern today?**
The idea still holds (separate data logic from UI), but is expressed with hooks: a `useXyz` hook for logic and a pure component for presentation. This improves testability and Storybook usage.

**Q47. How would you structure a large React codebase?**
Feature-based (vertical slice) structure beats type-based:
```
src/
  app/            # providers, routing, store setup
  features/
    claims/
      api/ components/ hooks/ state/ types.ts index.ts   # public API
  shared/         # ui kit, utils, hooks
```
Rules: features expose a public API through `index.ts`, no deep imports across features (enforce with ESLint boundaries), shared code has no feature dependencies, and colocate tests and styles.

**Q48. Monorepo and micro-frontends: when?**
**Monorepo** (Nx/Turborepo/pnpm workspaces) for shared design system, tooling, and atomic changes. **Micro-frontends** (Module Federation, single-spa, iframes, or web components) only when multiple independent teams must deploy independently. They cost you shared dependencies, consistency, performance, and complexity. Prefer a modular monolith frontend unless the organizational need is real.

**Q49. How do you build a design system?**
Tokens (color, spacing, typography), headless primitives (Radix, React Aria) for accessibility, themed components, Storybook as living docs, visual regression tests (Chromatic), semantic versioning, and a contribution model. Style via CSS Modules, CSS-in-JS with zero runtime (vanilla-extract, Panda), or Tailwind. Be aware of the runtime cost of traditional CSS-in-JS and its SSR/RSC friction.

**Q50. Error handling strategy across the app?**
Error boundaries at route and feature level, a global handler plus reporting (Sentry), typed API errors, retry and backoff in the data layer, user-friendly fallback states, and logging with correlation IDs that match backend traces.

---

## 8. Routing, SSR & Next.js

**Q51. CSR vs SSR vs SSG vs ISR vs streaming?**
- **CSR:** the browser renders everything. Simple, but slow first paint and poor SEO without extra work.
- **SSR:** HTML rendered per request. Good SEO and first paint, but server cost and TTFB.
- **SSG:** pre-rendered at build. Fastest, for content that changes rarely.
- **ISR:** SSG with background revalidation.
- **Streaming SSR:** send HTML progressively with Suspense boundaries.
Choose per route.

**Q52. What is hydration, and what causes hydration mismatches?**
Hydration attaches event listeners and state to server-rendered HTML. Mismatches occur when the server and client render differently: `Date.now()`, `Math.random()`, `window`-dependent output, locale or timezone differences, and invalid HTML nesting. Fix by rendering the dynamic part after mount (in an effect), `useId`, and deterministic output.

**Q53. How does client-side routing work?**
The router intercepts navigation, updates the URL via the History API, and renders the matching component tree without a full reload. Know data-loading routers (React Router loaders/actions, Next.js layouts), nested routes, lazy routes, and protected routes (redirect in a loader or middleware, never only in the UI).

---

## 9. Testing

**Q54. Testing strategy for React apps?**
Testing-trophy approach:
- **Static:** TypeScript and ESLint.
- **Unit:** pure functions and hooks (`renderHook`).
- **Integration (the bulk):** React Testing Library, testing behavior as the user sees it (roles, labels, text), not implementation details.
- **E2E:** Playwright or Cypress for critical flows.
- **Visual regression:** Storybook with Chromatic.
- **API mocking:** MSW, so tests use realistic network behavior.

**Q55. How do you test a component with async data?**
Render with providers, mock the network with MSW, then use `findBy*` or `waitFor` to assert on the result. Query by role and accessible name (`getByRole('button', { name: /submit/i })`), and use `userEvent` over `fireEvent`.

**Q56. What should you not test?**
Implementation details (internal state, private methods, CSS class names), third-party library internals, or snapshot-test everything (brittle, rubber-stamped).

---

## 10. Security

**Q57. How does React help with XSS, and where is it still vulnerable?**
JSX escapes interpolated values by default. Risks: `dangerouslySetInnerHTML` (sanitize with DOMPurify), `javascript:` URLs in `href`, untrusted data in `eval`/`new Function`, and vulnerable dependencies. Add a strict **CSP**.

**Q58. Where do you store auth tokens?**
Prefer **httpOnly, Secure, SameSite cookies** (not readable by JS, so XSS can't steal them), with CSRF protection (SameSite plus anti-CSRF tokens for cookie-based sessions). `localStorage` is exposed to XSS. For SPAs, use the BFF pattern or Authorization Code + PKCE, with short-lived access tokens and refresh rotation.

**Q59. How do you handle authorization in the frontend?**
UI gating (hiding buttons, routes) is for **UX only**. The real enforcement is on the server for every request. Drive the UI from claims and permissions returned by the API, and never trust the client.

**Q60. Other frontend security practices?**
Dependency auditing (npm audit, Dependabot), subresource integrity for CDN scripts, no secrets in the bundle (anything in the bundle is public), CORS configured server-side, and clickjacking protection (`frame-ancestors`).

---

## 11. TypeScript with React

**Q61. How do you type props, events, and children?**
```tsx
type ButtonProps = {
  variant?: 'primary' | 'secondary';
  onClick?: React.MouseEventHandler<HTMLButtonElement>;
} & React.ComponentPropsWithoutRef<'button'>;

function Button({ variant = 'primary', ...rest }: ButtonProps) {
  return <button data-variant={variant} {...rest} />;
}
```
Use `React.ReactNode` for children, `ComponentProps<typeof X>` to derive types, and avoid `React.FC` unless you need it.

**Q62. How do you write a generic component?**
```tsx
type ListProps<T> = { items: T[]; render: (item: T) => React.ReactNode; getKey: (i: T) => string };
function List<T>({ items, render, getKey }: ListProps<T>) {
  return <ul>{items.map(i => <li key={getKey(i)}>{render(i)}</li>)}</ul>;
}
```

**Q63. Discriminated unions for component state?**
Model impossible states as unrepresentable:
```ts
type State =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: Claim[] }
  | { status: 'error'; error: Error };
```

---

## 12. Tricky / Output-Based Questions

**T1. What does this log, and why?**
```jsx
const [count, setCount] = useState(0);
const handle = () => {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
  console.log(count);
};
```
Logs the old value (`0`), and the final state is `1`. State is a snapshot per render, and all three calls use the same `count`. Use `setCount(c => c + 1)` to get `3`.

**T2. Why does this loop forever?**
```jsx
useEffect(() => { setData([...]) }, [data]);
```
The effect sets state, which changes `data`, which re-runs the effect. Fix the dependencies or the logic. Don't derive state in an effect.

**T3. Stale closure:**
```jsx
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, []);
```
`count` is captured as `0` forever, so it ticks to 1 and stays there. Fix with `setCount(c => c + 1)`.

**T4. Why is a memoized child still re-rendering?**
Because the parent passes a new object, array, or function reference each render. Stabilize with `useMemo`/`useCallback`, or move the value out.

**T5. Does `setState` update immediately?**
No. It schedules an update; the new value is visible on the next render. It is batched.

**T6. Why can't you call hooks conditionally?**
Hook identity depends on call order. A conditional call shifts the order between renders and mismatches state slots.

**T7. What happens to state when you change a component's `key`?**
React unmounts the old instance and mounts a new one, **resetting state**. It is a deliberate technique to reset a form when the entity ID changes.

**T8. What is the output order?**
```jsx
useEffect(() => console.log('effect'));
useLayoutEffect(() => console.log('layout'));
console.log('render');
```
`render`, then `layout`, then `effect`.

---

## 13. System Design Questions

**S1. Design a large-scale dashboard (real-time charts, filters, thousands of rows).**
- **Data:** server-side filtering, sorting, and pagination; TanStack Query with keys from filters; WebSocket or SSE for live updates, throttled and batched into state.
- **Rendering:** virtualized table, memoized cells, canvas/WebGL for heavy charts, and `useDeferredValue` on filter input.
- **State:** filters in the URL, server data in the query cache, and minimal local UI state.
- **UX:** skeletons via Suspense, error boundaries per widget, and independent widget loading (no single waterfall).
- **Ops:** Web Vitals monitoring, bundle budgets, and feature flags.

**S2. Design an infinite-scroll feed.**
Cursor-based pagination (not offset), an `IntersectionObserver` sentinel, `useInfiniteQuery`, list virtualization with dynamic row heights, scroll restoration, optimistic likes and comments, deduplication by ID, and a "new posts" banner instead of auto-inserting.

**S3. Design a multi-step form / claim submission wizard.**
A state machine (XState or reducer) for steps, per-step schema validation (Zod) shared with the backend contract, draft autosave (debounced, idempotent), file upload with direct-to-storage pre-signed URLs and progress, resumable drafts, accessibility (focus management, error summaries), and analytics on drop-off.

**S4. Design a micro-frontend platform for 5 teams.**
First challenge the need. If justified: Module Federation or runtime composition behind a shell; shared design system and singleton React; a contract for events, routing, and auth; independent CI/CD; performance budgets; and shared observability. Call out the costs: duplicated dependencies, inconsistent UX, and harder local development.

**S5. Design a real-time collaborative editor (conceptual).**
Transport via WebSockets; conflict resolution via CRDTs (Yjs) or OT; presence and cursors in ephemeral state; an optimistic local-first model; offline queue and sync; and undo/redo scoped per user.

**S6. How would you migrate a legacy app (jQuery/Angular.js/class components) to modern React?**
Strangler-fig approach: embed React islands in the legacy page, migrate route by route, put shared contracts (API client, auth, design tokens) first, keep both running behind feature flags, add characterization tests before moving, and measure each step. Avoid a big-bang rewrite.

---

## 14. Behavioral Questions for Architects

- **How do you decide between building in-house and adopting a library?** Evaluate maintenance, bundle size, community and bus factor, licensing, accessibility, and cost of switching. Wrap third-party APIs behind your own interface.
- **How do you enforce consistency across teams?** Lint rules, templates and generators, a design system, architectural decision records (ADRs), code owners, and CI gates (type check, tests, bundle size, a11y checks).
- **How do you manage frontend performance as a team practice?** Performance budgets in CI, RUM (real-user monitoring), dashboards for Web Vitals, and performance reviews in the definition of done.
- **How do you handle technical debt?** Make it visible, tie it to business risk, budget a share of each sprint, and refactor incrementally behind tests.
- **Describe a hard trade-off you made.** Use a structure: context, options, decision criteria, decision, outcome, and what you would do differently.
- **How do you mentor seniors and juniors differently?** Juniors get pairing and clear conventions; seniors get ownership, design review, and context on the "why".

---

## 15. Quick Revision Cheat Sheet

- Render is a **pure function of props and state**. Side effects go in event handlers or effects.
- **State is a snapshot.** Use the functional updater when the next value depends on the previous one.
- Effects: declare all dependencies, **clean up**, and prefer not to use them for derived state or data fetching.
- Optimize in this order: **structure and composition, then measure, then memoize**.
- Server state is not client state. Use a cache library.
- Lists: stable keys, virtualization for large sets.
- Split code at routes. Stream and server-render where SEO or first paint matters.
- Test behavior, not implementation. Mock the network with MSW.
- Never trust the client for authorization. Prefer httpOnly cookies, and sanitize any raw HTML.
- Architecture answers should always include a **trade-off** and when you would *not* use the pattern.

---

### How to use this file
1. Read each answer aloud once, then close the file and re-explain it in your own words.
2. For every question, add a short real-world example from your own experience.
3. Practice the output-based questions (section 12) without running the code first.
4. Rehearse the system design questions (section 13) on a whiteboard in 30 to 40 minutes each.
