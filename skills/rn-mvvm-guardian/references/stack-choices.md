# Stack choices: pick your libraries, keep the boundaries

MVVM in React Native does not require any particular library. For each concern
there's a menu — what matters is that the chosen library lives **inside one
layer**, and that the rest of the app depends on that **layer's contract**, not
the library. That is what keeps the app faithful *and* keeps the library
swappable.

## The menu (per concern)

| Concern | Options | Lives in (the boundary) | What the rest of the app depends on |
|---|---|---|---|
| **Navigation** | React Navigation · expo-router · custom | a `navigation/` facade (`appNavigation`, `useRouteId`, `useScreenTitle`) | the facade's intent-named methods — never the router |
| **Server state / data fetching** | TanStack Query (formerly React Query) · SWR · RTK Query · hand-rolled hooks · none (call services directly from the VM) | `queries/` + `mutations/` | a feature-defined neutral shape — the **canonical** one is `useProductsData`'s `{ items, isLoading, isRefreshing, error, refetch }` ([`triad-example.md`](triad-example.md) §5); the exact fields are the screen's to define (add `loadMore`/`hasMore` for pagination, drop `isRefreshing` if there's no pull-to-refresh) |
| **HTTP client** | fetch · axios · ky | `services/` + a `client` factory | typed data + domain errors — never the HTTP type |
| **Client state** | Zustand · Redux Toolkit · MobX · Jotai (Recoil — unmaintained, see below) · Context | `stores/` (or observables, in MobX), accessed via the lib's granularity (selector / `observer` / atom) | a slice/observable contract; teardown in a coordinator |
| **Input/validation** | hand-rolled parsers · zod · yup | `parsers/` (input→value) | pure functions returning domain-ready values |
| **Form orchestration** | react-hook-form · Formik · manual VM state | the **ViewModel** (field state, errors, submit); uses `parsers/` to clean values + a `service`/`mutation` to submit | the VM exposes `values`, `errors`, `onChangeX`, `onSubmit`, `status` to the View |
| **Persistence** | MMKV · AsyncStorage · SecureStore | a storage adapter (behind the store, or backing the query cache) | the persisted shape — never the storage API |
| **Device / native capabilities** | expo-camera · vision-camera · expo-location · expo-notifications · react-native-permissions · expo-file-system · biometrics | a `services/` adapter (or a device port) behind an abstraction | typed results + domain errors — never the native module's API |
| **Styling / theming** | StyleSheet · NativeWind · Tamagui · Restyle · styled-components | the **View** + a `shared/theme` (tokens, dark mode) | theme tokens/components — Views style; VMs never see styling |
| **Observability** | analytics · logging · Sentry/Crashlytics | a thin `services/` reporter, called from the **VM** (intent) | a typed `track(event)`/`report(error)` contract — never fired from a View |

> The VM/View never import a library from this table. They import the **layer**.
> (Form libraries are the one nuance: the VM may own `useForm`, but keep the View
> passive — pass it ready `errors`/handlers rather than wiring the form lib's
> `Controller` in the View; domain-rule validation stays in the model/parser.)
>
> **`queries/` + `mutations/` is a *conceptual* boundary, not a mandated folder.**
> With TanStack/SWR it's hooks in `queries/`; with **RTK Query** it's an API slice
> colocated with the Redux store; with **Apollo/urql** the client *is* the cache.
> Whatever the shape, the rule is unchanged: the VM consumes a neutral hook, not the
> lib's result type.

## How swappable is each? (honest)

- **Navigation — very.** It's a pure facade (DIP). Swap React Navigation ↔
  expo-router by rewriting the facade + the route definitions; ViewModels and
  Views don't change.
- **HTTP — high.** Confined to the client factory + `services/` (+ the
  interceptor wiring). Swap fetch ↔ axios ↔ ky by rewriting those; the error
  classification (which knows the HTTP error type) moves with it. Layers above
  see plain data + domain errors.
- **Server state — high, with one caveat.** The query/mutation hooks are
  isolated, but VMs often consume the *library's* result shape (e.g. React
  Query's `isFetchingNextPage`/`hasNextPage`). To make it fully swappable, have
  the VM depend on a **feature-defined neutral hook** — e.g. a paginated
  `useProductsData()` returning `{ items, isLoading, error, loadMore, hasMore }`
  (the canonical non-paginated shape is `{ items, isLoading, isRefreshing, error,
  refetch }` — [`triad-example.md`](triad-example.md) §5; add/drop fields per the
  screen) — and let that hook wrap whichever library you chose. Then swapping the
  library is invisible to the VM. (The paginated end-to-end slice — neutral hook,
  VM `ready` fields, the View's footer spinner — is [`triad-example.md`](triad-example.md) §18.)
  - **Suspense is an alternative *mechanism*, not a different contract.** With
    React 18 `Suspense` (TanStack Query's `useSuspenseQuery`, or `use(promise)`),
    the data hook **suspends** instead of returning `{ isLoading }`, and an error
    boundary catches failures instead of an `error` field. The VM then drops the
    `loading` (and often `error`) variant and models only `empty`/`ready` — the
    discriminated-union contract is unchanged, just shorter. **Suspend at the
    query/neutral-hook boundary, never from the VM**; the VM still returns a
    `status`-keyed contract, so the View and tests don't change. Pick one mechanism
    per screen — `{ isLoading, error }` fields *or* Suspense — don't mix both for the
    same data.
- **Client state — medium.** Stores are isolated, but the *access idiom* (e.g.
  `useStore((s) => s.x)`) appears at every consumer. Swapping Zustand ↔ Redux
  touches each call site's selector. Keeping selectors small and centralizing
  derived state limits the blast radius. **MobX shifts the axis** — an observable
  store can sit closer to the VM; keep the observability inside `stores/` and
  expose a contract (not the raw observable) so the VM stays swap-tolerant.
- **Device / native capabilities — high.** Confined to a `services/` adapter (or a
  device port); the VM sees typed results + domain errors, so swapping the native
  lib (e.g. one camera/location lib for another) is a one-adapter change. Platform/
  dimension state (`Platform`, `useWindowDimensions`) belongs in a UI hook, not the VM.

**Rule of thumb:** a library swap should touch **one layer**. If a swap forces
changes in Views or ViewModels, a boundary is leaking — add the neutral
adapter/facade for that concern.

## Per-library quick reference (lib → layer → what the VM sees → the leak → how to catch it)

The same boundaries, instantiated for the libraries developers most often ask
about. "Grep" is a low-recall triage probe (adapt the store/hook name); where a
grep is unreliable, it says **read-review**.

| Library | Layer / boundary | What the VM depends on | The leak (red flag) | Catch it |
|---|---|---|---|---|
| **TanStack Query** (React Query) | server-state in `queries/` | a neutral hook (`useXData`) returning `{ items, isLoading, error, refetch, … }` | `useQuery`/`hasNextPage`/`isFetchingNextPage` used **in a VM** | `grep -rEn "from ['\"]@tanstack/react-query['\"]" --include='*ViewModel*' src` → none |
| **SWR** | server-state in `queries/` | the same neutral hook | `useSWR` (or its `mutate`/`isValidating`) **in a VM** | `grep -rEn "from ['\"]swr['\"]" --include='*ViewModel*' src` → none |
| **RTK Query** | server-state = an **API slice colocated with the Redux store** (this *is* the server-state layer) | a feature-defined neutral hook wrapping the generated `useXQuery` | the generated hook's raw result shape (`isFetching`, `originalArgs`, …) reaching a VM | wrap the generated hook in `queries/`; `grep` the generated hook name in `*ViewModel*` → none |
| **Redux Toolkit** (client state) | a slice in `stores/`, read via `useSelector` | a selector's return (a minimal slice) + intent-named dispatch | `useSelector((s) => s)` (whole store); cross-concern teardown living in a slice/reducer | `grep -rEn "useSelector\(\s*\(s\w*\)\s*=>\s*s\w*\s*\)" src` (whole-state selector) → none; teardown → a coordinator |
| **Zustand** | a store in `stores/`, read via a selector | `useStore((s) => s.x)` — one slice | `useXStore()` with no selector (whole-store subscription) | `grep -rEn 'use\w*Store\(\)' src` → none |
| **MobX** | an observable store in `stores/`; the **VM may itself be observable** | a contract method/computed (not the raw observable); `observer` wraps the View | the raw observable handed to the View; UI logic in the store | **read-review** — keep observability inside `stores/`/VM, expose a contract |
| **Jotai** (Recoil†) | atoms/selectors (in `stores/` or feature-level) | a *specific* atom/selector via `useAtomValue(atom)` / `useRecoilValue` | subscribing to a broad atom when a derived/split atom exists (ISP) | **read-review** — prefer focused/derived atoms |
| **Context API** (client state) | a provider in `stores/` or `shared/` | the typed value via a small selector-style hook (`useAuth()`), not the raw context | business logic in the provider; passing one big value object that re-renders every consumer | **read-review** — split contexts; keep the provider thin |

Whatever the library, the invariant is identical: it lives in **one** layer, the VM
sees a **neutral contract**, and a swap touches that layer only.

> † **Recoil is no longer actively maintained** (Meta wound the project down). The
> boundary advice still applies to an existing Recoil codebase, but for a **new**
> project prefer **Jotai** (the same atom model, actively maintained) — and migrate
> Recoil to it when practical. Because atoms sit behind the same `stores/` boundary,
> that swap is a one-layer change.

### Client state with no extra library — Context API, in code

The lowest-dependency option still obeys the boundary: the provider holds the state,
a small **selector-style hook** is the neutral contract the VM consumes (never the
raw context), and business logic stays out of the provider.

```tsx
// shared/stores/AuthContext.tsx — a thin provider; no business logic here
const AuthContext = createContext<{ token: string | null; setToken: (t: string | null) => void } | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [token, setToken] = useState<string | null>(null);
  const value = useMemo(() => ({ token, setToken }), [token]); // memo → consumers don't re-render on unrelated renders
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

// the neutral contract the VM depends on — derived state computed, not stored (ISP: expose only what's needed)
export function useIsAuthenticated(): boolean {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useIsAuthenticated must be used within AuthProvider');
  return ctx.token !== null;
}
```
> Context's swap caveat (the per-library table): one big value object re-renders every
> consumer — split contexts by concern and expose narrow hooks (`useIsAuthenticated`,
> not `useAuthContext`) so a VM subscribes to the minimum, exactly as a Zustand/Redux
> selector would. Teardown spanning concerns still goes to a coordinator, not the provider.

## Concerns beyond the core menu (where they plug in)

The same rule applies — confine the library to one layer behind a neutral contract.
**Several are now worked end-to-end in [`integration-recipes.md`](integration-recipes.md)**
(GraphQL §4, real-time §2, offline-first/sync §3, deep linking §5, security §6); the rest
plug in as below:

- **GraphQL** (Apollo · urql · Relay) — the client fuses HTTP **and** the
  server-state cache. Treat it as the *server-state* boundary: keep the client +
  generated types in one place and have the VM depend on a **neutral feature hook**
  (`useProductsData`) wrapping `useQuery`/`useMutation`, not on Apollo types. A
  pure-GraphQL app has no separate `services/` HTTP layer — the client is it.
- **Real-time** (WebSocket · SSE · GraphQL subscriptions) — a `services/` **stream
  adapter** that pushes into the query cache or a store; the VM consumes the same
  neutral shape it would for a fetch. Connect/teardown is an effect in the VM (or a
  coordinator).
- **Offline-first / sync** (WatermelonDB · RxDB · Replicache · query-cache
  persistence) — a **persistence adapter** + a **sync coordinator**; the VM reads
  the same domain types. Sync/conflict logic never reaches a View or VM.
- **Deep linking & typed route params** — owned by the **navigation facade**;
  validate inbound params with a `parser` at the boundary before the VM trusts them.
- **Security** (token lifecycle, refresh, cert pinning, PII) — token logic lives
  behind the auth service / the `AuthBridge` port; secrets in secure storage; never
  log PII (a rule for the observability reporter).
- **Side-effect orchestration** (redux-saga · redux-observable/epics) — these *are*
  the coordinator/effect layer for a Redux app: a cross-concern flow
  (login → fetch profile → navigate) lives in a saga/epic, never in a reducer or a
  View. The VM dispatches an intent and reads the result via a selector; domain
  rules stay in the model — the saga only orchestrates.
- **State machines** (XState · its `@xstate/react` hooks) — a legitimate
  *implementation* of the VM's discriminated-union `status`, not a new layer. Keep
  the machine inside the ViewModel (or a hook the VM owns) and expose the **same
  neutral contract** (`status` + ready values) to the View; the View still only
  branches on `status`.
- **Feature flags / remote config** (LaunchDarkly · Firebase Remote Config ·
  Statsig · a home-grown service) — a `services/` adapter (optionally feeding a
  store) behind a neutral contract; the VM reads a **ready** boolean/value (via a
  selector or a neutral hook) and folds it into "what to show". The View never reads
  the flag SDK and never branches on a raw flag — gating UI is the VM's decision,
  the View just renders the resolved branch.

## Choosing for a new project (by capability, never by default)

The skill picks nothing for you. Choose each library by the *capability you actually
need* — not by habit or a default — then keep it behind its boundary so the choice
stays reversible. The menu above names the instances; below is only how to reason
about which capability a screen actually requires:

- **Server state** → first ask whether you need a server-state *lifecycle* at all
  (caching, dedup, pagination, refetch, shared invalidation). If not, the VM can call
  a `service` and hold the result in local state — the `service` is still the
  abstraction over the HTTP client, so DIP holds; this is the one documented case
  where a VM touches the data layer with no `queries/` hook between them. When that
  lifecycle becomes a real need, introduce a server-state library (any of the menu
  options) and the VM depends on the neutral hook, not the service.
- **Navigation** → decide by the routing model you want (file-based vs
  imperative/config-based); any router satisfies the boundary. Whichever you pick,
  hide it behind the `navigation/` facade.
- **Client state** → decide by the subscription granularity you need — a
  minimal-dependency provider, selector-based stores, observables, and atoms all
  honour ISP. Reach for heavier middleware/devtools only when a concrete need (not
  habit) appears.
- **HTTP** → the built-in `fetch` covers the baseline; a client library adds
  interceptors / retry / ergonomics — adopt one only if you need those capabilities.
  Either way it lives only in the client factory + `services/`.
- **Persistence** → decide by what the data *is*: secrets → a secure keystore; small
  hot prefs → a fast KV store; large/low-priority blobs → async storage; *server*
  state → back the query cache, never a client store. The storage API stays behind
  the persistence adapter.

Whatever you choose, the skill's only job is to ensure it sits inside its boundary,
so the app stays MVVM-faithful and the choice stays reversible.
[`worked-examples.md`](worked-examples.md) is **one** concrete, copy-and-adapt
instantiation (Expo + expo-router + TanStack Query + Zustand + axios) — an example to
adapt, never a recommended default.
