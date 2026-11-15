---
name: react-19
description: "React 19.x and the current frontend stack (Nov 2026) that pre-2026-cutoff models get wrong. Use when writing React components, forms, data fetching with TanStack Query, or configuring Vite. Covers React 19.2/19.3 APIs (useEffectEvent, Activity, ViewTransition), React Compiler 1.0 with Vite 8, React Router v8, and TanStack Query v5 syntax."
---

# React 19.x ecosystem (Nov 2026)

Current: React **19.3**, React Compiler **1.0 stable** (Oct 2025), Vite **8.x** (Rolldown bundler), React Router **v8**, TanStack Query **v5** (no v6 exists — the npm "6.4.1" is `@tanstack/svelte-query`, not React). CRA was sunset Feb 2025 — Vite is the official non-framework path.

## Stale → current

| Stale | Current |
| --- | --- |
| Create React App scaffolding | `npm create vite@latest` (Vite 8 templates) |
| `forwardRef` wrapper | Deprecated — `ref` is a regular prop since 19.0 |
| `useFormState` | Renamed **`useActionState`** (19.0) |
| `onSubmit` + `preventDefault` | `<form action={fn}>` is the preferred API (works in plain SPAs, no server needed) |
| `eslint-disable exhaustive-deps` for non-reactive logic | **`useEffectEvent`** (19.2, stable) |
| Hidden-but-mounted tab content hacks | **`<Activity>`** (19.2): `mode="visible"/"hidden"`, state kept |
| React Router v6 style + `react-router-dom` imports | v7 merged Remix; v8 (June 2026): import from **`react-router`** (DOM bits like `RouterProvider` from `react-router/dom`); `react-router-dom` removed |
| `useQuery(key, fn, options)` | v5: **object arg only** |
| `isLoading` for initial load | `isPending` |
| `cacheTime` | `gcTime` |
| `keepPreviousData: true` | `placeholderData: keepPreviousData` (imported) |
| Vite = esbuild + Rollup | Vite 8 (Mar 2026): **Rolldown** (Rust) unified bundler, esbuild/rollupOptions auto-converted |
| React Compiler RC/beta | **1.0 stable**; pin exact versions (`--save-exact`) |

## React 19.2/19.3 — new APIs

```tsx
// useEffectEvent (19.2): non-reactive logic, no deps, no disables
const onOpen = useEffectEvent(() => toast('connected'));
useEffect(() => {
  socket.on('open', onOpen);
  return () => socket.off('open', onOpen);
}, [socket]);

// <Activity> (19.2): hide without unmounting state, effects unmount properly
<Activity mode={tab === 'details' ? 'visible' : 'hidden'}>
  <DetailsPanel />
</Activity>
```
19.3 (Sep 2026): `<ViewTransition>` and Fragment refs (`<Fragment ref>`, `.focus()`/`.getClientRects()`) went stable. 19.1+: Owner Stacks, `useId` format changed.

## React Compiler 1.0 + Vite 8

```js
// npm i -D babel-plugin-react-compiler @rolldown/plugin-babel
// NOTE: @vitejs/plugin-react@6+ removed the inline `babel` option — the old
// react({ babel: { plugins: ['babel-plugin-react-compiler'] } }) no longer works.
import { defineConfig } from 'vite';
import react, { reactCompilerPreset } from '@vitejs/plugin-react';
import babel from '@rolldown/plugin-babel';

export default defineConfig({
  plugins: [react(), babel({ presets: [reactCompilerPreset()] })],
});
```
`create-vite` ships a compiler-enabled template. Opt out per component with `"use no memo"`. Verify via the "Memo ✨" badge in React DevTools. Lint rules live in `eslint-plugin-react-hooks` presets now (`eslint-plugin-react-compiler` is discontinued).

## TanStack Query v5

```tsx
import { keepPreviousData, useQuery } from '@tanstack/react-query';

const { data, isPending } = useQuery({
  queryKey: ['items', q],
  queryFn: () => api.items(q),
  placeholderData: keepPreviousData,
});
```
Also v5: `throwOnError` (was `useErrorBoundary`), flag renamed `isPlaceholderData`, `onSuccess/onError/onSettled` removed from `useQuery` (kept on `useMutation`), infinite queries require explicit `initialPageParam`, error type defaults to `Error`. Modern imperative API: `queryClient.query()` (`fetchQuery`/`prefetchQuery` deprecated, slated for removal in v6).

## Forms in a plain Vite SPA (no server framework)

```tsx
const [error, formAction, isPending] = useActionState(
  async (_prev: string | null, fd: FormData) => {
    try { await api.signup(String(fd.get('email'))); return null; }
    catch (e) { return e instanceof Error ? e.message : 'Failed'; }
  },
  null,
);

<form action={formAction}>      {/* no onSubmit, no preventDefault */}
  <input name="email" required />
  <button disabled={isPending}>{isPending ? '…' : 'Sign up'}</button>
  {error && <p role="alert">{error}</p>}
</form>
```
`'use server'` actions still require a server framework (Next/RR framework mode) — but the `action={fn}` prop pattern above is fully client-side. React Hook Form (7.x) remains current for complex/validated forms.

## Version-pin security note

Never pin react **19.0.0 / 19.1.1 / 19.2.0** — critical RCE-class RSC vulnerabilities fixed in 19.0.1 / 19.1.2 / 19.2.1 (Dec 2025).

## Sources

- https://react.dev/blog/2025/10/01/react-19-2 (19.2) · https://react.dev/blog/2026/09/09/react-19-3 (19.3)
- https://react.dev/blog/2025/10/07/react-compiler-1 · https://react.dev/learn/react-compiler/installation
- https://vite.dev/blog/announcing-vite8 (2026-03-12) · https://www.npmjs.com/package/vite (8.3.8)
- https://remix.run/blog/react-router-v8 (2026-06-17) · https://reactrouter.com/changelog (8.5.1)
- https://tanstack.com/query/latest/docs/framework/react/guides/migrating-to-v5 · https://www.npmjs.com/package/@tanstack/react-query (5.107.1)
- https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components
