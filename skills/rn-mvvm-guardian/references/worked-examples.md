# Worked examples: conformance greps, lint recipes, and migration playbooks

A **concrete instantiation** of the stack-agnostic contract for *one* popular
stack: Expo + expo-router + TanStack Query + Zustand + axios. The recipes below
(greps, the ESLint feature-boundary lint, the `AuthBridge` inversion, the
React-Query/Zustand specifics, the worked screen→feature migration) are
copy-pasteable as-is for that stack — but the **patterns** generalize. On a
different stack, keep the pattern and swap the specifics: see
[`stack-choices.md`](stack-choices.md) for where each library plugs in and how to
write the equivalent grep/facade/port for your navigation, server-state, HTTP, and
client-state libs. Read [`mvvm-and-scaling.md`](mvvm-and-scaling.md) for the
stack-neutral contract these examples instantiate. For the triad in code (the
Model→…→View→Screen slice and the VM contract), see
[`triad-example.md`](triad-example.md).

> **Not on this stack? [section 10](#10-the-same-recipes-on-a-contrasting-stack-bare-rn--react-navigation--redux-toolkit--fetch) re-derives every recipe on a contrasting one** (bare React
> Native + React Navigation + Redux Toolkit + `fetch`, no Expo, no server-state lib) —
> so you can see the greps, the I/O service, the facade, and the coordinator change
> *imports only*, never the boundaries. The triad itself is shown across MobX / RTK
> Query / Redux in [`triad-advanced.md`](triad-advanced.md) [section 15](triad-advanced.md#15-the-same-triad-on-another-stack--a-mobx-class-vm-an-rtk-query-swap-and-a-redux-client-state-vm).

> **About the imports in these snippets.** Third-party imports (`axios`, `zustand`,
> `@tanstack/react-query`, `expo-router`, …) are real packages. App-internal imports
> (`client`, `toAppError`, `transformProduct`, `Product`, `secureStore`, `navigationRef`,
> `refreshSession`, the Redux `authSlice`/`store`, …) are the **same assumed helpers
> catalogued in [`triad-crosscutting.md` section 23](triad-crosscutting.md#23-referenced-helpers--primitives-assumed-not-re-implemented-here)** or thin stand-ins you implement per your design — this is
> illustrative TypeScript (adapt the imports to your libraries), not a runnable project.
> What matters is the **boundary** each file sits behind, not the helper bodies.

---

## 1. Conformance greps (run from the project root)

Portable across GNU and BSD/macOS grep: `-E` everywhere (no BRE `\|`), and
`--include` instead of a `**` glob (which needs bash `globstar`). Import
patterns use `['\"]` so they match either quote style (single or double).

> ⚠️ **These greps are low-recall triage, not proof.** They catch the frequent
> patterns (`toFixed`, dates, a couple of template shapes) — not all formatting
> (e.g. `` `Total: ${x}` ``, `Intl`, dayjs slip through). "expect: none" means "no
> hits of *these* patterns", not "no formatting". Confirm by reading.

```bash
# ViewModels must not import the framework or render JSX:
grep -rEn "from ['\"]react-native['\"]" --include='*ViewModel*' src   # expect: none
# Views must not format (*View.tsx only — *View* would also scan *ViewModel*):
grep -rEn '\.toFixed\(|toLocaleDateString|`\$\$\{|`@\$\{' --include='*View.tsx' src   # expect: none
# axios confined to transport:
grep -rEn "from ['\"]axios['\"]" src   # expect: only api/ + services/ (a query retry predicate may import axios error types/guards only, never axios values)
# router behind the facade, not in VMs/Views:
grep -rEn "from ['\"]expo-router['\"]" src   # expect: only navigation/ + app/ (not ViewModels/Views)
# Views depend only on the VM contract — never the data/navigation layers directly:
grep -rEn "from ['\"][^'\"]*/(services|queries|mutations|stores|coordinators|navigation)(/|['\"])" --include='*View.tsx' src   # expect: none
# Zustand selectors, not whole-store:
grep -rEn 'useAuthStore\(\)|useStore\(\)' src   # expect: none (always use a selector)
```

For **feature-based** projects, the feature boundary must hold:
```bash
# deep cross-feature imports (illegal): anything deeper than a feature root
# (feature names may contain digits/hyphens, e.g. user-profile, so [a-z0-9-]+)
grep -rEn '@/features/[a-z0-9-]+/[a-zA-Z]' src   # expect: none (only dev test-utils, by exception)
# shared/ must not import a feature:
grep -rn '@/features' src/shared             # expect: none (except a sanctioned dev test helper)
# intra-feature imports must be relative, not the alias to self:
grep -rn '@/features/<name>' src/features/<name>   # expect: none
```

Gate every change: `bunx tsc --noEmit` = 0 (or `npx`/`yarn`/`pnpm` on a non-bun project), `<lint>` = 0, `<test>` green, boundary greps clean.

---

## 2. Feature-boundary lint recipe (ESLint flat config, no extra plugins)

```js
// every src file: deep cross-feature imports are an error
{ files: ["src/**/*.{ts,tsx}"], rules: { "no-restricted-imports": ["error", {
  patterns: [{ group: ["@/features/*/*"],
    message: "Import a feature only through its public API: '@/features/<name>'." }] }] } },
// shared/** only: importing any feature is an error
{ files: ["src/shared/**/*.{ts,tsx}"], rules: { "no-restricted-imports": ["error", {
  patterns: [{ group: ["@/features/**"],
    message: "shared/ must not import from features/." }] }] } },
// dev test-utils may reference feature public types — opt it out explicitly:
{ files: ["src/shared/test-utils/**/*.{ts,tsx}"], rules: { "no-restricted-imports": "off" } },
// Views (*View.tsx) additionally must not import the data/navigation layers — a View
// depends only on the VM contract. NOTE: in flat config a later object's
// `no-restricted-imports` REPLACES (not merges with) an earlier one for matching files,
// so this block repeats the cross-feature pattern to keep BOTH protections on Views:
{ files: ["src/**/*View.tsx"], rules: { "no-restricted-imports": ["error", {
  patterns: [
    { group: ["@/features/*/*"],
      message: "Import a feature only through its public API: '@/features/<name>'." },
    { group: ["**/services/**", "**/queries/**", "**/mutations/**", "**/stores/**", "**/coordinators/**", "**/navigation", "**/navigation/**"],
      message: "A View depends only on the VM contract — move data/navigation access to the ViewModel." },
  ] }] } },
// ViewModels (use*ViewModel.ts) must stay UI-free: no react-native, no rendering, no
// presentational components. (Pairs with the grep in section 1; the lint is what
// survives turnover.) The VM imports `react` itself (useState/useMemo) — only
// `react-native` and UI components are forbidden:
{ files: ["src/**/use*ViewModel.ts"], rules: { "no-restricted-imports": ["error", {
  paths: [{ name: "react-native",
    message: "A ViewModel is UI-free — no react-native import. Move rendering to the View." }],
  patterns: [
    { group: ["@/features/*/*"],
      message: "Import a feature only through its public API: '@/features/<name>'." },
    { group: ["@/shared/components/**", "**/components/**"],
      message: "A ViewModel must not import UI components — it exposes a contract the View renders." },
  ] }] } },
// Domain/Model layer (models/) must not import the infrastructure layers — the Model
// depends on NOTHING (it's the innermost layer). This is the "domain ↛ infrastructure"
// rule: a model file reaching for a service/query/store/the HTTP client has leaked I/O
// into the domain. (RN has no "repositories"; the service layer is the analogue — and a
// service importing a screen/View is the same inversion, caught by the *View.tsx block
// above plus this one.)
{ files: ["src/**/models/**/*.{ts,tsx}"], rules: { "no-restricted-imports": ["error", {
  paths: [
    { name: "axios", message: "The Model layer is pure domain — no HTTP client. I/O lives in services/." },
    { name: "react-native", message: "The Model layer is framework-free — no react-native." },
  ],
  patterns: [
    { group: ["**/services/**", "**/queries/**", "**/mutations/**", "**/stores/**", "**/navigation/**", "**/views/**", "**/screens/**"],
      message: "The Model (domain) layer must not depend on infrastructure/UI — it is the innermost layer and depends on nothing." },
  ] }] } },
```
Because intra-feature imports are relative, the first rule only ever fires on an
illegal cross-feature import.

> **Why these rules earn their keep.** A grep catches a leak *if someone runs it*; an
> ESLint rule fails the build the moment a ViewModel reaches for `react-native` or a
> View imports `queries/`. **Expected violations are the point** — when the rule fires,
> it has caught exactly the boundary erosion the contract exists to prevent, and the
> fix is always to move the import to the layer that owns it (rendering → the View,
> data/navigation → the ViewModel). The rules above need **no extra plugin**; for
> path-based layering at scale (e.g. forbidding the model/domain layer from importing
> `services/`), `eslint-plugin-boundaries` or `eslint-plugin-import`'s
> `no-restricted-paths` express the same intent declaratively.

---

## 3. RN React-Query / Zustand specifics to verify
- The query `select` is for **cheap view-shaping/derived selection only** (wire→domain runs in the service, [section 5](#5-mapping-taxonomy-rn) above) and must be stable: infinite-query `select` wrapped in `useCallback`; single-query `select: mapFn` a module-level function.
- Gate id-based detail queries with `skipToken` (TanStack v5: `queryFn: id === null ? skipToken : () => fetchProduct(id)`) — the `parseRouteId` parser in [`triad-advanced.md`](triad-advanced.md) [section 14](triad-advanced.md#14-typed-route-params--validated-at-the-boundary-the-vm-never-imports-the-router) already guarantees a positive integer or `null`, so the gate is just the null-check, and `skipToken` both disables the query and narrows `id` to `number` (no `as` cast); capability queries gated on their param the same way. (And map `isLoading` from `q.isLoading`, **not** `q.isPending`, on a gated query — a skipped query stays `isPending` forever; see [`triad-example.md`](triad-example.md) [section 5](triad-example.md#5-neutral-feature-hook--wraps-the-server-state-lib-so-the-vm-never-sees-it).)
- `getNextPageParam`: returns the next `skip` (e.g. `skip + limit`) while `skip + limit < total`, else `undefined` — it returns the next param, not a boolean.
- Mutations invalidate the right key on settle; optimistic updates snapshot + rollback on error (worked end-to-end in [`triad-advanced.md`](triad-advanced.md) [section 17](triad-advanced.md#17-mutations--the-write-path-behind-a-neutral-hook-invalidation--optimistic-update), behind a neutral `mutations/` hook).
- Zustand: selectors only; `persist` `partialize` excludes runtime-only flags; auth derived from token presence (a selector), not a duplicated flag; secrets in secure storage, prefs in fast KV.

(List virtualization, memoized rows, and load-time derived display are stack-neutral
RN best practices — they live in `mvvm-and-scaling.md` [section 2](mvvm-and-scaling.md#2-conformance-checklist-the-keep-it-faithful-core) Hygiene, not here.)

---

## 4. The two hard inversions (feature-based)

**a) The `api ⇄ auth` cycle → an `AuthBridge` port.** The authenticated client
needs the token + refresh; auth needs the client. Don't let `shared/api` import a
feature. Instead:
- `shared/api/interceptorPort.ts` defines an `AuthBridge` interface (`getAccessToken`, `getRefreshToken`, `setAccessToken`, `refresh`, `onAuthFailure`) + `attachAuthInterceptors(client, bridge)` — depends only on the HTTP lib + the interface.
- `features/auth/api/authInterceptor.ts` implements the bridge from the auth store/service/session and exposes `setupAuthInterceptors()`.
- `app/_layout` calls `setupAuthInterceptors()` at boot. Direction: auth → shared, never the reverse.

**b) Navigation: shell vs feature-owned.** Shell transitions (which screen is
"login"/"home") live in `shared/navigation` (`goToLogin`, `goToAuthenticatedHome`).
Feature-owned deep routes (`openProduct`, `openPost`, …) live in each feature's
`navigation.ts`. Result: no cross-feature *navigation* — features navigate within
themselves or to a shell route.

---

## 5. Mapping taxonomy (RN)
- **`transformers/`** `transform*`: **wire DTO → domain** (rename, *representation* derivations like cents-from-dollars, fallbacks; **domain rules** like `finalPrice`/`totalTimeMinutes` live in the model — the transformer only *invokes* them, never computes them here; runs once **at the fetch edge — the service's `queryFn`**, so the query/neutral hook already yields domain types; see [`triad-example.md`](triad-example.md) [section 2](triad-example.md#2-service--the-only-layer-that-touches-io-classifies-errors)) **and domain → request** for create/update — both directions live together because they change for the same reason (the wire contract). *Don't* re-run wire→domain in the query `select` (that's for cheap view-shaping only, below).
- **`formatters/`** `to*`: domain → view-item/string; `to*Message` for error→copy. Primitives (`formatPrice`/`formatRating`/`formatDate`/`formatCount`) in `shared/formatters`.
- **`parsers/`** `parse*`: user input → value (e.g. `parseQuantity("abc",{min,max})→min`); primitives (`parseInteger`/`parseDecimal`/`clamp`) in `shared/parsers`.
- View-item types co-locate with their producing `to*`. More generally: **no reflexive `types.ts`** — keep a type next to its owner (DTO with its transformer, view-item with its formatter), and extract to a dedicated file only when it's shared across feature files or it dominates the file (see [`conventions.md`](conventions.md#glossary)).
> Rung-1 note: a minimal app needs **fewer** mapping folders, not a different home for them. `to*` always lives in `formatters/`; add `transformers/` only once the wire shape diverges from the domain, and `parsers/` only once forms appear. The prefix means exactly one thing on every rung (`transform*`=wire↔domain, `to*`=domain→view, `parse*`=input→value) — so a folder is *absent* in a tiny app, never *repurposed*.

---

## 6. Component & hook tiers (feature-based)
- **The View orchestrates; cohesive blocks get extracted.** The View composes the screen and delegates — an independent visual block with **its own behavior or a clearly separate responsibility** becomes its own component. Extract by **cohesion/responsibility, not line count**: a block earns a file when it's a nameable unit, not because the View got long; and don't shatter one cohesive screen into throwaway components either (premature abstraction, flagged like the oversized View — see [`conventions.md`](conventions.md#view-composition--orchestrate-extract-by-cohesion)).
- **Components:** screen-local (`screens/<S>/components/`) → feature-level (`features/<f>/components/`, 2+ screens or public) → shared (`shared/components/`, 2+ features).
- **Hooks:** the **ViewModel is a hook** (`use<Screen>ViewModel`); *reusable* hooks live feature-level (`features/<f>/hooks/`, even if one screen uses it — a hook is feature logic) → shared (`shared/hooks/`, when a 2nd feature consumes it). Asymmetry: presentational components may be screen-local; reusable logic lives at feature level.
- **UI hooks hold no data** (state/dimensions/scroll/animation/focus) — data orchestration stays in the ViewModel/queries. A pure UI hook is the one kind of hook the **View consumes directly** (the VM never mediates it; see [`triad-crosscutting.md`](triad-crosscutting.md) [section 21](triad-crosscutting.md#21-animations--gestures--a-ui-hook-that-holds-no-data-never-the-viewmodel)). Typical patterns: a password-visibility toggle, a quantity stepper (paired with `parseQuantity`), a responsive-columns hook, an avatar image→initials fallback, an expandable "read-more" text hook.
- **Layers are optional:** a feature has only the folders it needs. A *capability feature* (e.g. `comments`) has no `screens/`/`navigation`; a *thin feature* (e.g. `profile`) has no `services/`/`queries/`/`models/`. A missing folder is not a defect — don't flag it against a fixed template.

---

## 7. Migration playbook — screen-based → feature-based (worked)
Phased; each phase keeps tsc/lint/tests green.
1. **Foundation:** create `shared/` (api/components/config/constants/formatters/hooks/navigation/stores/theme/test-utils); add the boundary-lint rules; invert the api⇄auth cycle (AuthBridge); split shell vs feature navigation.
2. **Per feature** (one at a time): create `features/<name>/` with **only the layers it needs** (models/transformers/formatters/parsers/services/queries/mutations/components/hooks/screens/navigation — as applicable; capability/thin features have fewer); convert imports to relative (intra) + `@/shared` (cross-cutting); add the public `index.ts`; re-export its screens from `app/`. Keep its tests co-located.
3. **Capability features** (no screen) are fine — expose a component/hook via `index.ts` (e.g. a comments list other features compose).
4. **Compose** across features only through public roots; keep the graph acyclic.
5. **Verify** after each feature: tsc/lint/tests + the boundary greps.

For **feature → modular monolith** and **→ micro-frontend**, see the generic
`mvvm-and-scaling.md` [section 4](mvvm-and-scaling.md#4-migration-playbooks-mechanical-moves-per-step) (promote features to packages with build-enforced
boundaries; then independent build/deploy + runtime composition).

---

## 8. The boundaries in code (this stack: expo-router · Zustand · TanStack · axios)

The recipes the contract describes in prose, here as copy-pasteable code for this
stack. On another stack, keep the shape and swap the library (see
[`stack-choices.md`](stack-choices.md)).

**a) Navigation facade + the neutral focus hook.** The only place `expo-router` is
imported (besides `app/` route files). VMs call these by intent, never the router.

```ts
// shared/navigation/appNavigation.ts — shell transitions (intent-named)
import { router } from 'expo-router';

export const appNavigation = {
  goToLogin: () => router.replace('/login'),
  goToAuthenticatedHome: () => router.replace('/(tabs)'),
  goBack: () => router.back(),
};
```

```ts
// shared/navigation/useScreenFocusEffect.ts — neutral focus hook (wraps the router's hook)
import { useFocusEffect } from 'expo-router';
import { useCallback, useRef } from 'react';

// VMs consume THIS for "refetch on focus" — never expo-router's useFocusEffect directly.
// A ref holds the latest callback while the effect identity stays stable, so the effect
// doesn't re-register every render even when the VM passes an inline arrow.
// (Don't write `useFocusEffect(useCallback(onFocus, [onFocus]))` — that memo is a no-op:
// useCallback(fn, [fn]) just returns fn, stabilizing nothing.)
export function useScreenFocusEffect(onFocus: () => void): void {
  const cb = useRef(onFocus);
  cb.current = onFocus;
  useFocusEffect(useCallback(() => { cb.current(); }, []));
}
```

On **React Navigation** the facade is the swap target — same intent methods, same
neutral focus hook, only the imports change (this is the swappability claim in code):

```ts
// shared/navigation/appNavigation.ts (React Navigation) — same SHAPE as the expo-router version above
import { navigationRef } from './navigationRef'; // created next to the NavigationContainer at app root

export const appNavigation = {
  goToLogin: () => navigationRef.reset({ index: 0, routes: [{ name: 'Login' }] }),
  goToAuthenticatedHome: () => navigationRef.reset({ index: 0, routes: [{ name: 'Tabs' }] }),
  goBack: () => navigationRef.goBack(),
};
// useScreenFocusEffect: identical body, swap the import to '@react-navigation/native'.
```
VMs and Views are untouched by the swap — **only this file changes.**

**b) Client store + selector (Zustand).** A pure state container: no navigation, no
cache, no server state. Auth is **derived** from token presence (a selector), never a
duplicated boolean.

```ts
// shared/stores/useAuthStore.ts
import { create } from 'zustand';

interface AuthState { token: string | null; setToken: (t: string | null) => void }

export const useAuthStore = create<AuthState>((set) => ({
  token: null,
  setToken: (token) => set({ token }),
}));

export const selectIsAuthenticated = (s: AuthState) => s.token !== null; // derived, not stored
// consumer subscribes to a slice, not the whole store (ISP):
//   const isAuthed = useAuthStore(selectIsAuthenticated);
```

**c) Logout coordinator.** Teardown that spans concerns (store + server-cache +
navigation) lives **here**, not in the store. Holds no state of its own; the VM calls it.

```ts
// shared/coordinators/logoutCoordinator.ts
import type { QueryClient } from '@tanstack/react-query';
import { useAuthStore } from '@/shared/stores/useAuthStore';
import { appNavigation } from '@/shared/navigation/appNavigation';

export function makeLogout(queryClient: QueryClient) {
  return () => {
    useAuthStore.getState().setToken(null); // 1. clear client state
    queryClient.clear();                     // 2. clear server-state cache
    appNavigation.goToLogin();               // 3. navigate to the shell route
  };
}
```

The VM must **not** acquire the `QueryClient` itself (that would import the
server-state lib into the VM — the result-shape/lib leak red flag). Expose the
coordinator through a **neutral hook that lives in the coordinator layer** — the one
place allowed to import the lib — so the VM consumes a bare `() => void`:

```ts
// shared/coordinators/useLogout.ts — the neutral seam the VM depends on
import { useQueryClient } from '@tanstack/react-query'; // imported HERE (coordinator layer), never in a VM
import { useMemo } from 'react';
import { makeLogout } from './logoutCoordinator';

export function useLogout(): () => void {
  const queryClient = useQueryClient();
  return useMemo(() => makeLogout(queryClient), [queryClient]); // VM does: `const logout = useLogout()`
}
```

**d) The `AuthBridge` port (the `api ⇄ auth` inversion, [section 4a](#4-the-two-hard-inversions-feature-based) in code).** Defined by the
side that *needs* auth (`shared/api`), implemented by the feature — so `shared` never
imports a feature.

```ts
// shared/api/interceptorPort.ts — depends only on the HTTP lib + this interface
import type { AxiosInstance } from 'axios';

export interface AuthBridge {
  getAccessToken: () => string | null;
  getRefreshToken: () => Promise<string | null>; // secrets live in async secure storage (Expo SecureStore / Keychain) — never read synchronously
  setAccessToken: (t: string) => void;
  refresh: () => Promise<void>;
  onAuthFailure: () => void;
}

export function attachAuthInterceptors(client: AxiosInstance, bridge: AuthBridge): void {
  client.interceptors.request.use((config) => {
    const token = bridge.getAccessToken();
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
  });
  // response interceptor: on 401 → bridge.refresh() once; on repeat → bridge.onAuthFailure()
}
```

```ts
// features/auth/api/authInterceptor.ts — auth IMPLEMENTS the port (direction: auth → shared)
import { client } from '@/shared/api/client';
import { attachAuthInterceptors, type AuthBridge } from '@/shared/api/interceptorPort';
import { useAuthStore } from '@/shared/stores/useAuthStore';
import { secureStore } from '@/shared/persistence/secureStore'; // refresh token = a secret
import { refreshSession } from './refreshSession';

const bridge: AuthBridge = {
  getAccessToken: () => useAuthStore.getState().token,
  getRefreshToken: () => secureStore.get('refreshToken'), // adapter wraps Expo SecureStore (async) → returns Promise; secrets never in the client store
  setAccessToken: (t) => useAuthStore.getState().setToken(t),
  refresh: refreshSession,
  onAuthFailure: () => useAuthStore.getState().setToken(null),
};

export const setupAuthInterceptors = () => attachAuthInterceptors(client, bridge);
// app/_layout.tsx calls setupAuthInterceptors() once at boot.
```

---

## 9. Testing the non-triad layers

The triad's own tests are in [`triad-example.md`](triad-example.md) [section 9](triad-example.md#9-tests--the-fake-vm-view-test--the-renderhook-vm-test). The boundary
layers are tested too:

- **Service** — mock the `client` (`jest.mock('@/shared/api/client')`); assert it
  returns domain types on success and maps transport failures to the right `AppError`
  (e.g. a 503 → `{ kind: 'server', status: 503 }`). The `toAppError` classifier is a
  pure function tested input→output.
- **Coordinator** — call `makeLogout(queryClient)()` with a fake `queryClient` + a real
  store; assert the store was cleared, `queryClient.clear` was called, and the nav
  facade was invoked (spy on `appNavigation.goToLogin`). No rendering needed.
- **Navigation facade** — `jest.mock('expo-router')`; assert each intent method calls
  the router with the expected route. The neutral param hook is tested with
  `renderHook` + mocked `useLocalSearchParams`, asserting the parsed value.
- **Store (client state)** — no rendering: drive the store's setters and assert the
  derived selector — `useAuthStore.getState().setToken('x');
  expect(selectIsAuthenticated(useAuthStore.getState())).toBe(true)`. Reset the store
  between tests. The selector itself is a pure function, also tested input→output.
- **Error boundary** — render a child that throws inside `<AppErrorBoundary>`; assert
  the fallback renders and the reporter was called (`jest.mock` the reporter). Then
  assert recovery: invoking the retry handler clears `hasError` and re-renders the
  children. (Silence the expected `console.error` React emits for a caught render error.)

---

## 10. The same recipes on a contrasting stack (bare RN · React Navigation · Redux Toolkit · fetch)

[sections 1–9](#1-conformance-greps-run-from-the-project-root) instantiate one popular stack. [`triad-advanced.md`](triad-advanced.md) [section 15](triad-advanced.md#15-the-same-triad-on-another-stack--a-mobx-class-vm-an-rtk-query-swap-and-a-redux-client-state-vm)
already proves the **triad** is stack-agnostic (MobX / RTK Query / Redux). This
section proves the **operational recipes** are too — the greps, the I/O service, the
coordinator — by re-deriving them for a deliberately different stack: **bare React
Native** (no Expo), **React Navigation**, **Redux Toolkit** for client state, and
**`fetch`** with **no server-state library**. Every boundary is identical; only what
sits inside each layer changes. Copy *this* set if it matches your stack; otherwise
read it as the template for writing your own.

**a) Conformance greps for this stack** (the [section 1](#1-conformance-greps-run-from-the-project-root) probes, re-pointed at these libs):
```bash
# ViewModels must not import the framework — stack-neutral, unchanged:
grep -rEn "from ['\"]react-native['\"]" --include='*ViewModel*' src   # expect: none
# Views must not format — stack-neutral, unchanged:
grep -rEn '\.toFixed\(|toLocaleDateString|`\$\$\{|`@\$\{' --include='*View.tsx' src   # expect: none
# fetch confined to services/ — `fetch` is a global (no import to grep), so this is a
# read-review probe: any fetch( outside services/ is a transport leak:
grep -rEn '\bfetch\(' src --include='*.ts' --include='*.tsx' | grep -v '/services/'   # expect: none
# React Navigation behind the facade, not in VMs/Views:
grep -rEn "from ['\"]@react-navigation/(native|native-stack)['\"]" src   # expect: only navigation/ + the navigator setup
grep -rEn '\buseNavigation\(' --include='*ViewModel*' --include='*View.tsx' src   # expect: none (VMs use the facade)
# Views depend only on the VM contract — stack-neutral, unchanged from section 1:
grep -rEn "from ['\"][^'\"]*/(services|queries|mutations|stores|coordinators|navigation)(/|['\"])" --include='*View.tsx' src   # expect: none
# Redux whole-store selector (ISP leak) — same probe as the per-library table:
grep -rEn 'useSelector\(\s*\(s\w*\)\s*=>\s*s\w*\s*\)' src   # expect: none — always a slice selector
```

**b) The I/O service with `fetch`** — the same contract as [section 2](triad-example.md#2-service--the-only-layer-that-touches-io-classifies-errors) (`Promise<Product[]>`
plus a classified `AppError`); only the client changes. Two differences from axios:
`fetch` does **not** reject on a non-2xx status (so the service checks `res.ok`
itself), and `toAppError` here is the **fetch-aware** classifier — error classification
always travels with the HTTP client (see `stack-choices.md` swappability). It is
**idempotent** (an `AppError` in → the same `AppError` out), so the explicit HTTP-status
throw flows cleanly through the single `catch`. We always `throw` a classified
`AppError`, never a raw literal (which `no-throw-literal` would reject):
```ts
// features/products/services/productService.ts (fetch — no axios)
import { transformProduct, type ProductDTO } from '../transformers/transformProduct';
import { toAppError } from '@/shared/api/toAppError'; // fetch-aware classifier; idempotent on AppError
import type { Product } from '../models/product';

export async function fetchProducts(opts?: { signal?: AbortSignal }): Promise<Product[]> {
  try {
    const res = await fetch('/products', { signal: opts?.signal });
    if (!res.ok) throw toAppError({ status: res.status }); // HTTP status → domain error (no literal thrown)
    const data: { products: ProductDTO[] } = await res.json();
    return data.products.map(transformProduct);            // wire → domain at the edge (same as section 2)
  } catch (e) {
    throw toAppError(e);                                   // transport / parse error → domain error (idempotent on the line above)
  }
}
```
With **no server-state lib**, the VM owns fetch + cancellation directly — that is the
[`triad-advanced.md`](triad-advanced.md) [section 12](triad-advanced.md#12-alternative--no-server-state-library-the-vm-owns-fetch--cancellation) VM verbatim (it already returns the [section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union)
`ProductsVM` contract, so the View/Screen/test are unchanged). Add TanStack/RTK Query
later and only the `queries/` hook appears.

**c) Navigation facade + client store** — same *shape* as [section 8a](#8-the-boundaries-in-code-this-stack-expo-router--zustand--tanstack--axios)/[section 8b](#8-the-boundaries-in-code-this-stack-expo-router--zustand--tanstack--axios), different imports.
The React Navigation facade is the [section 8a](#8-the-boundaries-in-code-this-stack-expo-router--zustand--tanstack--axios) swap target (`navigationRef.reset(...)`,
`goBack()`), and the Redux client-state slice + selectors are
[`triad-advanced.md`](triad-advanced.md) [section 15c](triad-advanced.md#15-the-same-triad-on-another-stack--a-mobx-class-vm-an-rtk-query-swap-and-a-redux-client-state-vm) (the slice is the boundary; the VM
subscribes via a slice selector, never `useSelector((s) => s)`). The VM calls the
facade's intent methods and reads a selector — exactly as in the canonical stack.

**d) Logout coordinator with no `QueryClient`** — the [section 8c](#8-the-boundaries-in-code-this-stack-expo-router--zustand--tanstack--axios) teardown, minus the
server-state cache (there's no TanStack here; server data lives in the Redux store, or
in an RTK Query API slice). Teardown still lives in the coordinator, **never** a
reducer or a View:
```ts
// shared/coordinators/logoutCoordinator.ts (Redux, no TanStack)
import type { AppStore } from '@/shared/stores/store';
import { resetAuth } from '@/shared/stores/authSlice';
import { appNavigation } from '@/shared/navigation/appNavigation';

export function makeLogout(store: AppStore) {
  return () => {
    store.dispatch(resetAuth());            // 1. clear client state (a root-reducer reset clears every slice)
    // using RTK Query? also: store.dispatch(api.util.resetApiState()) — the server-cache lives in the store here
    appNavigation.goToLogin();              // 2. navigate to the shell route
  };
}
// Exposed to the VM through a neutral hook, exactly like section 8c's useLogout — but built on
// `useStore()` from react-redux instead of `useQueryClient()`. The VM still calls
// useLogout() and sees only a `() => void`; the store/dispatch wiring never reaches it.
```

The lesson: a **grep, a facade, a coordinator, a neutral hook** exist on *every* stack;
only their imports change. The boundaries — not the libraries — carry the recipes. If
swapping a library forced a change in a VM or a View, a boundary would be leaking; the
fix is always the neutral adapter/facade for that concern, never a change to the triad.

---

## 11. Common adoption mistakes (and the boundary each one breaks)

The frequent ways a faithful start drifts. Each is a single-layer fix — the table maps
the symptom to the rule and the section that shows the correct shape.

| Mistake (what you'll see in review) | Rule broken | The fix | Worked in |
|---|---|---|---|
| A formatter reads the locale from a **global i18n singleton** (`i18n.t(...)` inside a pure `to*`) | formatter purity / hidden dependency | thread `locale`/`t` from the VM as an argument | [`triad-crosscutting.md`](triad-crosscutting.md) [section 19](triad-crosscutting.md#19-localization-i18n--the-locale-enters-through-the-vm-formatters-stay-pure) |
| The VM calls `useQueryClient()` / `new QueryClient()` to wire logout | server-state lib leaking into the VM | consume a neutral coordinator hook (`useLogout()`) | [section 8c](#8-the-boundaries-in-code-this-stack-expo-router--zustand--tanstack--axios) · [`triad-example.md`](triad-example.md) |
| `useStore()` / `useSelector((s) => s)` — whole-store subscription | ISP | subscribe to a slice selector | [section 1](#1-conformance-greps-run-from-the-project-root) grep · [section 3](#3-rn-react-query--zustand-specifics-to-verify) |
| A View computes `loading`/`empty` (`if (!data) …`) instead of branching on `status` | passive-View / the View decides | move the decision into the VM's discriminated `status` | [`triad-example.md`](triad-example.md) [sections 6–7](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union) |
| Inline copy / `accessibilityLabel` string literals in a View | copy centralization / i18n | source ready strings from the VM (via `t`) | [`triad-example.md`](triad-example.md) [sections 19–20](triad-crosscutting.md#19-localization-i18n--the-locale-enters-through-the-vm-formatters-stay-pure) |
| Animation/`useSharedValue` state placed in the ViewModel | VM imports no `react-native` | move it to a UI hook (or the View) | [`triad-crosscutting.md`](triad-crosscutting.md) [section 21](triad-crosscutting.md#21-animations--gestures--a-ui-hook-that-holds-no-data-never-the-viewmodel) |
| `useSuspenseQuery` (or `use(promise)`) called inside the VM | suspend at the query boundary, not the VM | suspend in `queries/`; the VM stays `status`-keyed | [`triad-crosscutting.md`](triad-crosscutting.md) [section 22](triad-crosscutting.md#22-suspense--suspend-at-the-query-boundary-never-from-the-viewmodel) |
| Server data hand-copied into a Zustand/Redux slice that mirrors the API | server state belongs to the query layer | keep it in `queries/`; derive, don't duplicate | `mvvm-and-scaling.md` [section 1](mvvm-and-scaling.md#1-layer-responsibilities-the-contract) |
| Building `stores/`/`queries/`/`parsers/` on day one "to be safe" | over-engineering (adoption ladder) | start at the section 0 quickstart; add a layer when a reason to change appears | `conventions.md` |
| A `mappers/` folder | the umbrella is not a folder | use `transformers/`/`formatters/`/`parsers/` | `mvvm-and-scaling.md` [section 1](mvvm-and-scaling.md#1-layer-responsibilities-the-contract) |
| A top-level `utils/` / `lib/` / `misc/` catch-all | SRP (Single Responsibility Principle) — a grab-bag carries many reasons to change | name each by its reason: a transformer/formatter/parser, a model use-case, or — if it passes the copy-paste test — a `shared/utils/` primitive (one fn per file) | `conventions.md` glossary |
| A `shared/utils/` fn that imports a domain type / API / UI | fails the copy-paste test — it's a named layer mislabeled | move it to the layer that owns the reason (transformer/formatter/parser/use-case) | `conventions.md` glossary |
| A feature `helpers/` imported by *another* feature, or growing domain/API/UI logic | local implementation detail leaking / becoming a layer in disguise | keep `helpers/` private to the feature; promote anything with domain/API/UI reasons-to-change to its named layer | `conventions.md` glossary |
| A `presenters/` folder | ambiguous ownership — the role is already split | domain→view-item is the **formatter**; "what to show" is the **ViewModel** | `triad-example.md` [section 4](triad-example.md#4-formatter--domain--view-item-pure-display-ready) · [section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union) |

The through-line: every mistake is one responsibility landing in the wrong layer. The
remedy is never "tidy the file" — it's "move the reason-to-change to the layer that owns
it," which is always a small, reversible move.
