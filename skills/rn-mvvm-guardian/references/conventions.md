# Conventions, glossary, and canonical structure

The reference card: the vocabulary, the naming rules, the folder trees, how to
model the VM contract, how much to build (so you don't over-engineer), and what
this material does **not** yet prescribe in depth. Pairs with
[`triad-example.md`](triad-example.md) (the same ideas in code) and
[`mvvm-and-scaling.md`](mvvm-and-scaling.md) (the contract).

---

## Glossary

- **Triad** — Model + View + ViewModel: the invariant core, the same on every rung.
- **Screen** — the per-screen *composition root* that wires the triad (calls the VM
  hook, passes its output to the View). Not part of the classic triad, but an
  invariant role here.
- **Rung** — a level on the scaling ladder (screen-based → feature-based → modular
  monolith → micro-frontend).
- **Facade** — a thin, intent-named API in front of a library (e.g. the navigation
  facade) so callers depend on *intent*, not the lib.
- **Port** — an interface defined by the side that *needs* a capability; the
  provider implements it. Inverts a dependency so a shared/lower layer needn't
  import a feature/higher one (e.g. the `AuthBridge` port in `worked-examples.md`).
- **Adapter** — the concrete implementation of a port, or the thin layer that turns
  a library's shape into the app's own contract (e.g. a `useProductsData` neutral hook).
- **Coordinator** — orchestrates a single cross-concern action (e.g. logout =
  clear store + invalidate cache + navigate). Holds no state.
- **Mapper** — umbrella term for the three pure mapping layers: **transformer**
  (wire↔domain), **formatter** (domain→view), **parser** (input→value). *"Mapper"
  is never a folder* — the folders are `transformers/`, `formatters/`, `parsers/`.
- **Non-folders (the catch-all smell)** — `utils/`, `helpers/`, `lib/`, `misc/`,
  `mappers/`, and `presenters/` are **not layers in this contract** and should not be
  folders: each hides *several* reasons to change, which is an SRP smell. Route their
  contents to the layer that owns the reason — wire↔domain → `transformers/`;
  domain→display → `formatters/`; input→value → `parsers/`; domain computation/rules →
  a **model use-case**; a genuinely reusable *pure primitive* (`assertNever`,
  `formatPrice`, `clamp`) → `shared/` (`shared/formatters`, `shared/parsers`, or a small
  `shared/<primitive>.ts`). If a helper fits **none** of these, that's a signal it's
  mislabeled — not that you need a `utils/`.
- **No `presenters/`** — the "presenter" role of classic MV\* is already split, with a
  clear owner each: domain→view-item is the **formatter** (`to*`), and *"what to show"*
  (the discriminated `status` + ready values) is the **ViewModel**. A `presenters/`
  folder would duplicate that ownership — the ambiguity the one-reason-to-change rule
  exists to prevent.
- **Capability feature** — a feature with no screen; exposes a component/hook other
  features compose (e.g. a comments list).
- **Thin feature** — a feature with no data layer of its own; reads another
  feature's data via that feature's public API.
- **Neutral hook** — a feature-defined hook that wraps a library and returns a
  lib-agnostic shape, keeping the ViewModel swappable. (It is a specific kind of
  **Adapter** — the hook-shaped one.) A neutral hook that wraps the **server-state**
  lib (`use<Thing>Data`) *is* the server-state boundary, so it lives in **`queries/`**
  — not `hooks/`, which is for UI/interaction hooks that hold no data.
- **Service** — the only layer that performs external I/O (the network client + the
  native/device APIs); returns domain types and classifies failures into domain errors.
- **Store** — the global *client*-state container: pure state, no navigation, no
  caching, no server state.
- **Persistence** — a durable-storage adapter (secure store / fast KV / async
  storage) that sits behind a store or backs the query cache; never imported by a
  VM/View.
- **DTO** — the wire/transport shape the API returns; lives with the transformer that
  consumes it, and is **never** used as a domain type.
- **Use-case (domain function)** — a pure function in the model layer that holds
  domain computation/rules (e.g. `computeCartTotal`); the VM *calls* it instead of
  *being* the business logic.

## Naming conventions

| Thing | Convention | Example |
|---|---|---|
| ViewModel hook | `use<Screen>ViewModel` | `useProductsViewModel` |
| ViewModel contract type | `<Screen>VM` | `ProductsVM` |
| View component / file | `<Screen>View` / `*View.tsx` | `ProductsView` |
| Screen component / file | `<Screen>Screen` / `*Screen.tsx` | `ProductsScreen` |
| Transformer | `transform*` | `transformProduct` |
| Formatter | `to*` (`to*Message` for errors) | `toProductItem`, `toErrorMessage` |
| Parser | `parse*` | `parseQuantity` |
| Validator (input→typed fault) | `validate*` | `validateEmail` |
| Neutral data hook | `use<Thing>Data` | `useProductsData` |
| Neutral mutation hook | `use<Action>` | `useToggleFavorite` |
| UI hook (holds no data) | `use<Behavior>` | `usePasswordVisibility` |
| Service function | `<verb><Noun>` | `fetchProducts` |
| Domain use-case (pure model fn) | `<verb><Noun>` | `computeCartTotal` |
| Store hook | `use<Domain>Store` | `useAuthStore` |
| Selector | `select<Slice>` | `selectIsAuthenticated` |
| Coordinator factory / its neutral hook | `make<Action>` / `use<Action>` | `makeLogout` / `useLogout` |
| Folders | plural, lowercase | `models/ services/ queries/` |
| Co-located test | `*.test.ts(x)` next to the unit | `transformProduct.test.ts` |

> The `<Screen>VM` **contract type lives in its own file** (`viewmodels/<Screen>VM.ts`),
> imported by *both* the ViewModel hook and the View — so the View never imports the
> hook, only the contract (see [`triad-example.md`](triad-example.md) §6). At Rung 1
> a tiny app may co-locate the type in the hook file; because the View imports it
> `type`-only (erased at compile time) there's no runtime coupling either way —
> split it into its own file once the feature grows.

## Canonical folder trees

A feature/app has **only the folders it needs** — a missing one is not a defect.

### Screen-based (Rung 1)
```
src/
  screens/
    Products/
      ProductsScreen.tsx
      ProductsView.tsx
      useProductsViewModel.ts
      components/            # screen-local presentational components
  models/                   # shared domain types + rules
  errors.ts                 # the domain-error taxonomy (the AppError union)
  services/                 # HTTP behind the client factory (+ device adapters)
  queries/ mutations/       # server-state hooks (optional)
  stores/                   # client state
  coordinators/             # cross-concern actions (e.g. logout); appears with the first one, not pre-created
  persistence/              # durable-storage adapters (secure store / fast KV), behind a store or backing the query cache
  transformers/ formatters/ parsers/   # app-level, domain-specific mapping
  navigation/               # the navigation facade
  shared/                   # components/hooks/constants/theme + reusable primitives (e.g. assertNever, the exhaustiveness guard)
                            #   shared/hooks/ → reusable UI hooks (use<Behavior>); the ViewModel is the per-screen hook
```
> **Where UI hooks live at Rung 1.** There are no features yet, so a *reusable* UI
> hook (`usePasswordVisibility`, a quantity stepper…) goes in `shared/hooks/`; a hook
> used by a single screen may be co-located in `screens/<S>/`. UI hooks hold **no
> data** — data orchestration stays in the ViewModel/queries. When you climb to
> feature-based, reusable hooks move to `features/<f>/hooks/` and graduate to
> `shared/hooks/` only when a second feature consumes them.
> **One mapping home at Rung 1.** Keep `transformers/`/`formatters/`/`parsers/`
> top-level here. The second tier — `shared/formatters/`, `shared/parsers/` for
> *reusable primitives* (`formatPrice`, `parseInteger`) used across areas — appears
> only once a primitive is actually reused; don't pre-split it on day one (that's the
> adoption ladder below). The triad example's `@/shared/formatters/formatPrice`
> import assumes that split has already happened; at Rung 1 it's simply
> `formatters/formatPrice`. **The same applies to the error taxonomy:** at Rung 1
> it's `errors.ts` at the `src/` root (so the worked slices' `@/shared/errors` is
> just `@/errors` / `errors`); it moves under `shared/` only when you climb to
> feature-based.

### Feature-based (Rung 2)
```
src/
  app/                      # route registration only (re-exports feature screens)
  features/
    products/
      models/ transformers/ formatters/ parsers/
      services/ queries/ mutations/
      hooks/ components/ viewmodels/ views/ screens/
      navigation.ts         # feature-owned deep routes
      index.ts              # PUBLIC API — the only cross-feature entry point
  shared/                   # cross-cutting; MUST NOT import a feature
    api/ components/ config/ constants/ coordinators/ errors.ts formatters/
    hooks/ i18n/ navigation/ parsers/ persistence/ stores/ theme/ test-utils/
    assertNever.ts          # + small reusable primitives (the exhaustiveness guard, etc.)
```

### Modular monolith (Rung 3a)
Features become **packages** in a workspace; the triad inside each is identical —
only the wall around it (a `package.json` + build-enforced boundaries) is new.
```
packages/
  products/                 # was features/products/ — same triad inside
    src/                    # models/ … viewmodels/ views/ screens/ navigation.ts
    index.ts                # public exports (the old feature public API)
    package.json            # name: @app/products; may depend ONLY on @app/shared
  shared/                   # was src/shared/ — now @app/shared (or @app/core)
    src/ index.ts package.json
app/                        # host app: composes packages, owns routing/shell
  src/ package.json
package.json                # workspace root (npm/yarn/pnpm workspaces, or Nx/Turborepo)
metro.config.js             # watchFolders + nodeModulesPaths; exactly ONE react/react-native
tsconfig.base.json          # path aliases kept in sync with the Metro resolver
```
Boundaries now hold at **build time**: a feature package imports only `@app/shared`
+ its own internals; cross-feature only through published entry points.

### Micro-frontend (Rung 3b)
Each unit is **independently built and deployed** and composed at runtime; the triad
inside a remote is still identical.
```
host/                       # the shell: provides react/react-native as singletons (externals)
  src/ App.tsx
  package.json              # + Re.Pack/Module-Federation config; loads remotes at runtime
remotes/
  products/                 # built + deployed on its own pipeline
    src/                    # same triad inside
    index.ts package.json
  checkout/ …
contracts/                  # the versioned seam between units
  types/                    # a shared TYPES package (the contract) + runtime validators at the boundary
```
`react`/`react-native` (and any lib holding native state — navigation/gesture/
reanimated) are **host-provided singletons**; units agree on versioned contracts;
OTA and federated remotes are two update channels that must agree (see
[`mvvm-and-scaling.md`](mvvm-and-scaling.md) §4).

## Modeling the VM contract (state)

Prefer a **discriminated union** keyed on `status` so illegal states can't be
represented (see [`triad-example.md`](triad-example.md) §6). Extend the variants as
the screen needs:

| Variant | Carries | When |
|---|---|---|
| `loading` | — | first load |
| `error` | `message`, `onRetry` | load failed, nothing to show |
| `empty` | `onRefresh` | loaded, zero items |
| `ready` | view data, handlers | loaded with data |

Model *secondary* states as **fields inside `ready`**, not new top-level variants,
so the data stays on screen: `isRefreshing` (pull-to-refresh), `isLoadingMore`
(pagination), `banner?: string` (partial/soft error). Reserve a top-level
`submitting` variant for forms whose whole screen is blocked during a mutation.

> **VM internals are free — the contract is what's fixed.** *How* the VM computes
> its state is its own business: `useState`, `useReducer` (idiomatic once transitions
> multiply), or a state machine (XState — see [`stack-choices.md`](stack-choices.md))
> all produce the same discriminated contract; the View can't tell which. **React
> Suspense for data** is compatible too: a Suspense-based query lets the component
> *suspend* during load, so the nearest `<Suspense fallback>` renders the spinner and
> the VM **drops the `loading` variant entirely** — it models only
> `error`/`empty`/`ready`. Either way the View only branches on `status`.

## Adoption ladder — build the fewest layers that carry a real reason to change

Over-building is the most common failure. Start minimal; add a layer only when a
**distinct reason to change** appears:

1. **Tiny app** — Model + View + ViewModel + Screen, one `service`, one `formatter`.
   No store, no queries, no parsers. The VM calls the service and holds the result
   in local state (the documented no-`queries/` case in `stack-choices.md`).
   **Copyable starting point:** the ~40-line quickstart in
   [`triad-example.md`](triad-example.md) §0.
2. **+ server-state needs** (cache / refetch / pagination) → add `queries/` +
   `mutations/` (or a neutral data hook); the VM now depends on the hook.
3. **+ shared client state** → add `stores/`; a cross-concern teardown → a `coordinator`.
4. **+ forms** → add `parsers/`.
5. **Several areas / a few teams** → climb to feature-based (see the migration
   playbook in `mvvm-and-scaling.md` §4).

The full layer vocabulary (the 11 layers in the contract table, plus the Screen
composition-root role) is the *complete* toolbox, not a checklist to instantiate on
day one.

### How much to build, by app size (the same ladder, read as a matrix)

A quick gut-check so "any porte" is concrete — match the *smallest* row that fits, and
let real pain (not this table) push you down a row:

| App / team | Rung | Build | Don't build (yet) |
|---|---|---|---|
| Prototype / 1–3 screens, one dev | screen-based | Model + View + ViewModel + Screen, one `service`, one `formatter` (the §0 quickstart) | queries, stores, parsers, transformers, coordinators, navigation facade |
| Small app / handful of screens, one small team | screen-based | + `queries/` (when caching/refetch is real) · `transformers/` (when wire ≠ domain) · `navigation/` facade · `parsers/` (when forms appear) | features/, build boundaries, packages |
| Growing app / several distinct areas, a few teams | feature-based | vertical `features/<f>/`, public `index.ts` per feature, the `no-restricted-imports` boundary lint, `shared/` | packages, independent builds |
| Large app / many areas needing enforced isolation, one build | modular monolith | feature **packages** in a workspace, build-enforced boundaries, Metro `watchFolders` + one RN instance | independent deploy pipelines |
| Platform / independently-deployed units, separate release pipelines | micro-frontend | independent build+deploy per unit, versioned contracts package, runtime composition | (the top of the ladder) |

The triad and the five rules are **identical on every row** — only *how files are
grouped* changes. Climbing or descending is a reorganization, not a rewrite.

## Integration concerns — worked in `integration-recipes.md`

These real RN concerns are all **compatible with the contract**, and each is now worked
end-to-end in [`integration-recipes.md`](integration-recipes.md) under the same
discipline — *confine the library to one layer behind a neutral contract*. Summaries here;
[`stack-choices.md`](stack-choices.md) names where each plugs in:

- **Observability** (analytics / logging / crash reporting) — fired from the VM
  (intent) or a thin service; never from the View → integration-recipes §1.
- **Real-time** (WebSocket / SSE / subscriptions) — a `services/` stream adapter
  feeding a query/store; the VM consumes a neutral shape → integration-recipes §2.
- **Offline-first / sync** (WatermelonDB / RxDB / Replicache, query-cache
  persistence) — a persistence adapter + a sync coordinator; sync logic never leaks
  into Views/VMs → integration-recipes §3.
- **GraphQL** (Apollo / urql / Relay) — fuses HTTP + cache; map "client+cache" onto
  the server-state boundary and keep the VM on a neutral hook → integration-recipes §4.
- **Deep linking & typed route params** — owned by the navigation facade; validate
  params with a `parser` at the boundary → integration-recipes §5 (+ [`triad-example.md`](triad-example.md) §14).
- **Security** (token lifecycle, cert pinning, PII) — token logic behind the auth
  service / `AuthBridge`; secrets in secure storage → integration-recipes §6.

Two further concerns are covered **outside** that file, because they change tooling, not the
triad:

- **New Architecture** (Fabric / TurboModules / JSI) — now the **default** in
  current React Native. It changes the *native* rendering/bridge layer, not the
  triad: a TurboModule/JSI-backed native capability sits behind a `services/` device
  adapter like any other native API, and animation worklets (e.g. Reanimated) live in a
  UI hook or the View — never the VM.
- **CI/CD & governance** — wire the gates (typecheck / lint / test / boundary) into
  CI + pre-commit; record architecture decisions as ADRs; assign ownership (e.g.
  `CODEOWNERS`) so the contract survives team turnover (see [`mvvm-and-scaling.md`](mvvm-and-scaling.md) §7).

## Assumed tooling

**Path alias (`@/`).** Every example imports through `@/…` (e.g. `@/shared/api/client`,
`@/features/products`). That alias is **not** built in — configure it once, in two
places that must agree (the type-checker *and* the Metro runtime):

```jsonc
// tsconfig.json — so `tsc` and the editor resolve "@/…"
{ "compilerOptions": { "baseUrl": "src", "paths": { "@/*": ["*"] } } }
```
```js
// babel.config.js — so Metro resolves "@/…" at bundle time
plugins: [['module-resolver', { root: ['./src'], alias: { '@': './src' } }]]
```
At Rung 3 the same mapping moves to `tsconfig.base.json` and must stay in sync with
the Metro resolver (see the modular-monolith tree above).

**Version floor.** The examples assume **React 18+** (`useId`, automatic batching),
**TanStack Query v5** for the server-state slices (the `isPending`/`isLoading`/
`isRefetching` semantics in §5–§6 — on v4 the names differ, but only the neutral
`queries/` hook changes), and a **current React Native** (New Architecture default).
Nothing here needs React 19, but the discriminated-union contract and the Suspense
note above are written against this floor; on older majors, treat the modern APIs as
illustrative and keep the boundaries.

**TypeScript**; **Jest + React Native Testing Library** (with `renderHook`) for
tests. `renderHook` ships **inside** `@testing-library/react-native` from **v12.4+**
(Dec 2023); on older majors it lived in the separate `@testing-library/react-hooks`
package — pin RNTL ≥ 12.4 (or import `renderHook` from the legacy package) so the VM
tests resolve. **End-to-end tests (Detox / Maestro) are orthogonal** to the triad — they
drive the assembled app through the UI and don't care about layer boundaries; this
contract governs the unit/contract level (pure layers input→output, VM via its
contract, View with a fake VM), and E2E sits above it unchanged.

**On a JS-only project**, keep the contracts as conventions and run **degraded
gates**: replace the `tsc --noEmit` type gate with **ESLint** (`eslint .` = 0) plus
runtime shape-checking where it matters (PropTypes on Views, or a runtime validator
like `zod` at the wire/parser boundary). The layer boundaries still enforce
mechanically — the `no-restricted-imports`/path-based lint rules and the conformance
greps are language-agnostic — and the Jest suite stays the behavior gate. Only the
*type-level* guarantees become review conventions instead of compiler-enforced: the
discriminated-union "illegal states unrepresentable", the "no `any`" rule, and the
fake-VM contract typing.

The same triad slice, degraded to JS — the validator moves the guarantee from
compile time to a runtime boundary, the `status` discriminant survives as a plain
string convention:

```js
// transformers/transformProduct.js — zod validates the wire shape the type system no longer can
import { z } from 'zod';
const ProductDTO = z.object({ id: z.number(), title: z.string(), price: z.number(), rating: z.number().optional() });
export const transformProduct = (raw) => {
  const dto = ProductDTO.parse(raw); // throws at the boundary instead of `any` leaking inward
  return { id: dto.id, name: dto.title, priceCents: Math.round(dto.price * 100), rating: dto.rating ?? 0 };
};

// views/ProductsView.jsx — PropTypes stand in for the contract type; the View still only branches on status
ProductsView.propTypes = { status: PropTypes.oneOf(['loading', 'error', 'empty', 'ready']).isRequired, /* … */ };
```
