# MVVM contract, conformance checklist, and the scaling ladder

React Native, **stack-agnostic** — these principles hold whatever navigation /
server-state / HTTP / client-state libraries you pick (see
[`stack-choices.md`](stack-choices.md) for where each plugs into the boundaries).
[`worked-examples.md`](worked-examples.md) specializes the checks and playbooks for
one concrete stack (Expo + expo-router + TanStack Query + Zustand + axios); the
principles here are shared across every stack.

---

## 1. Layer responsibilities (the contract)

| Layer | Owns | Must NOT |
|---|---|---|
| **Model** | domain types + domain rules; wire DTOs adjacent | import UI, HTTP client, or framework |
| **Transformer** | wire→domain (rename, structural fallbacks, *representation* derivations like cents-from-dollars; runs once on fetch); also domain→request for create/update; may aggregate sources | format for display; do I/O; **compute domain rules** (those live in the Model — the transformer only invokes them); know the View |
| **Formatter** (domain→view) | turn domain values into display strings / view-items; error→message; may **invoke** a domain rule from the Model (e.g. `isTopRated` → a badge) to shape the view-item | mutate data; do I/O; **compute** domain rules (those live in the Model — invoke, don't re-implement) |
| **Parser** (input→value) | clean user input into typed values, with fallbacks; may also **validate** — returning a typed *fault* (`'invalid' \| 'empty' \| null`), never the display string | know the View; do I/O; produce the user-facing message (that's the formatter) |
| **Service** | the only layer that touches external I/O — the network client *and* native/device APIs (camera, location, notifications, biometrics, filesystem, permissions); classify transport & native errors into domain errors | format; cache; be called from Views |
| **Query/Mutation** | caching, pagination, retry, invalidation (server-state lifecycle) | UI; formatting; be consumed outside a ViewModel |
| **Store** | global client state; pure state container | navigate; clear caches; hold server state |
| **Persistence** | a durable-storage adapter — secrets in secure store, prefs in fast KV; sits behind a store or backs the query cache | be imported by a VM/View; leak the storage API upward |
| **Coordinator** | orchestrate actions that span concerns (e.g. logout = clear store + invalidate server-cache + navigate); holds no state of its own | hold UI state; format; be imported by a View |
| **ViewModel** | screen state + behavior + "what to show" (ready strings/booleans + a **discriminated-union** `status`; see [`triad-example.md`](triad-example.md) §6) | import UI/render JSX; touch HTTP/router directly; format inline |
| **View** | render the branch the VM resolved; forward events | format (`toFixed`/dates/templating); decide error/empty/loading; call service/store/query/navigation |
| **Screen** | wiring: build the ViewModel, pass its output into the View as props | hold state, JSX layout, styles, or logic |

> The one rule everything follows: **each piece of code has exactly one reason
> to change, and lives in the layer that owns that reason.** When one edit forces
> three layers, a responsibility has leaked.

> **Note — the transformer carries both directions on purpose.** `wire→domain`
> and `domain→request` share the *same* reason to change — the wire contract — so
> when the API's shape moves they move together, in one layer. Split into separate
> files (`transformRequest`/`transformResponse`) if one grows unwieldy; the owning
> concern is still one.

> **Note — which MVVM this is: React / Passive-View MVVM, not the two-way-binding
> kind.** This is **React MVVM** — data flows ViewModel → View through props and
> events flow back through callbacks (React's *unidirectional* data flow) — **not**
> the two-way data-binding MVVM of .NET/WPF, Android (LiveData/DataBinding), or
> SwiftUI, where the View binds directly to ViewModel observables and may format
> through value converters. It is also a **stricter** View than even textbook MVVM:
> textbook MVVM lets the View format through value converters; here all formatting is
> pushed into `formatters/` and the ViewModel hands the View ready strings. This is a
> deliberate "passive View" stance (closer to MVP's Passive View), chosen for
> testability — the View needs no formatting logic to verify. MVVM does not
> *require* it; it's the opinion this contract enforces.

> **Note — "ready strings" vs "no inline formatting" are not in tension.** The VM
> produces display-ready values by *calling a formatter* and passing the result
> through; what it must not do is format *inline* (a `.toFixed`/date/template in
> the VM body). Delegating to a formatter = fine; doing it itself = a leak.

> **Note — a formatter *invokes* a domain rule, never *recomputes* it.** The badge
> decision is a domain rule (it lives in the Model); the formatter only calls it and
> shapes the view-item around the result. Re-deriving the rule inline duplicates
> domain logic into the presentation layer, where it will drift.
> ```ts
> // ✓ invokes the Model rule
> export const toProductItem = (p: Product): ProductItemUI =>
>   ({ id: p.id, title: p.title, badge: isTopRated(p) ? 'Top Rated' : null });
> // ✗ recomputes the rule in the formatter (domain logic leaking into presentation)
> //   const isBest = p.rating >= 4.5;
> ```

> **Note — the View never reads a parser fault; the ViewModel decides.** A parser
> returns a typed *value* or a typed *fault* (`'invalid' | 'empty' | null`) — never
> the display string. The **ViewModel** consumes that result and decides what to
> show: on a fault it maps to a ready message via a formatter; on success it includes
> the value in `ready`. The View only ever receives `errorMessage?: string` /
> `isError: boolean`, never the raw fault tag (it "decides nothing"). Worked in
> [`triad-example.md`](triad-example.md) §10.

> **Note — "mapper" is an umbrella, not a folder.** Where this contract says
> *mapper* it means the three pure mapping layers collectively
> (transformer/formatter/parser). The folders are `transformers/`, `formatters/`,
> `parsers/` — **never** a `mappers/` folder.

> **Note — where DTOs and device adapters sit.** The wire **DTO type** lives with
> the transformer that consumes it (or beside the model); it is not a domain type.
> **`services/` holds both** the network services *and* the native/device adapters
> (camera, location…) — both are "the layer that touches external I/O"; split into
> `services/api` vs `services/device` subfolders if it grows.

> **Note — "Store must NOT hold server state" means the *client* store.** A
> server-state library that caches inside the store/client *by design* (RTK Query's
> API slice living in the Redux store; Apollo's normalized cache) *is* the
> server-state layer — it just shares the store object. That is fine. The
> anti-pattern is a **hand-rolled client slice mirroring server data**, which drifts
> from the source of truth. See [`stack-choices.md`](stack-choices.md).

> **Note — model the VM contract as a discriminated union.** Make `status` the
> discriminant and let each variant carry only its own data, so illegal states
> (e.g. `loading` with data, `error` with items) are unrepresentable. Worked code:
> [`triad-example.md`](triad-example.md) §6; how to model secondary states
> (refreshing, load-more, partial error): [`conventions.md`](conventions.md).

### Layers are optional — a feature has only what it needs

Not every feature or screen has every layer. **A missing `queries`/`mutations`/
`services`/`models`/`screens` folder is NOT a defect** — judge a feature by its
responsibility, not a fixed template:

- **Capability feature** — no screen, no navigation, no tab; it exposes a
  component or hook that *other* features compose (e.g. a comments list).
- **Thin feature** — no service/query/model of its own; it reads another
  feature's data via that feature's public API (e.g. a profile reading the auth user).
- A read-only area needs no `mutations`; a static area needs no `queries`; a
  feature with no forms needs no `parsers`.

### Hooks

- A **ViewModel is itself a hook** (`use<Screen>ViewModel`) — the per-screen unit
  of state + behavior.
- **Reusable hooks** live at the **feature level** (`features/<f>/hooks/`) even
  when only one screen uses them, and graduate to **shared** (`shared/hooks/`)
  when a *second feature* consumes them.
- **Asymmetry:** reusable *logic* lives at feature level; presentational
  *components* tied to one screen may be screen-local.
- **UI hooks** (toggle/disclosure, keyboard, scroll, animation, expandable text,
  steppers…) hold **no data** — data orchestration stays in the ViewModel/queries.
- **The neutral *data* hook is different from a UI/reusable hook.** A hook that
  wraps the server-state lib (`use<Thing>Data`) *is* the server-state boundary and
  lives in **`queries/`**, not `hooks/`. `hooks/` is for the data-free UI hooks
  above (and other reusable non-server logic).

### Dependency injection & testing the ViewModel

The VM importing a facade/service module *is* the DIP boundary at the architecture
level — it depends on the abstraction's contract, not the concretion. For test-time
substitution, pick one seam:

- **`jest.mock('<facade/neutral-hook>')` — the default for a hook VM.** It keeps the
  production signature parameter-free (the canonical [`triad-example.md`](triad-example.md)
  §6/§9 shape) and is the lightest seam. Use it unless you need to swap the dependency
  *outside* a test runner.
- **An injectable hook param with real defaults** — when you also need runtime
  substitution (Storybook, a test harness app, multiple wirings). Shown below.
- **React Context** — for app-wide dependencies shared across many VMs.

The injectable-param seam, for a hook VM, is just a destructured default — production
calls it with no args; a test passes a fake:

```ts
export function useProductsViewModel({ useData = useProductsData } = {}): ProductsVM {
  const { items, isLoading, isRefreshing, error, refetch } = useData(); // ← swap in tests
  // …unchanged
}
```

The VM is **tested via its contract**, not its internals: a **hook VM** with
`renderHook` (RN Testing Library), since it's a hook, not a pure function; an
**observable/class VM** (MobX) by instantiating it with fake collaborators and
asserting the exposed contract after actions. Either way you assert the *contract the
View consumes*. The pure layers (transformers/formatters/parsers/rules) are the ones
tested input→output. The boundary layers are tested too — **service** (mock the
client; assert domain types + error classification), **coordinator** (fake
collaborators + spy on the facade), **navigation facade** (mock the router) — worked
in [`worked-examples.md`](worked-examples.md) §9.

### Feature composition (feature-based and up)

Features compose each other **only through public surfaces** (a feature's
exported component/hook), never deep imports. A capability feature contributes a
composable component/hook with no screen of its own. Keep the cross-feature graph
**acyclic and directional**.

### Behavior that spans the triad

- **Effects & lifecycle live in the ViewModel — never the View.** Because the VM is
  a hook, `useEffect` (fetch on mount, cancel on unmount) belongs there. **With a
  server-state lib** the fetch lifecycle is *owned by the query layer* (`useQuery`
  inside the neutral `queries/` hook), so the VM has no fetch `useEffect` at all — it
  just consumes the hook; the manual `useEffect` + `AbortController` is the
  *no-server-state-lib* path (see [`triad-example.md`](triad-example.md) §12). Either
  way, guard against races (an `AbortController` or the data lib's cancellation) and
  never start effects from a View. **Lifecycle hooks owned by the navigation library**
  (`useFocusEffect`/`useIsFocused`, for "refetch on focus") are the one exception:
  the VM consumes them through a **neutral hook the navigation facade exposes** (e.g.
  `useScreenFocusEffect(cb)`), never importing the router into the VM — the same
  rule as every other navigation call.
- **Coordinator vs ViewModel.** A VM orchestrates *its own screen*. The moment an
  action spans concerns that outlive the screen — multiple stores **and** the
  server-cache **and** navigation (logout, "switch workspace", account deletion) —
  extract it to a **coordinator** (`shared/coordinators/` or feature-level) the VM
  *calls*. Rule of thumb: if teardown touches state another screen also owns, it's
  a coordinator.
- **Keep the ViewModel thin — extract a use-case when domain logic grows.** The VM
  *orchestrates* (calls data + maps to "what to show"); it should not *be* the
  business logic. When a single screen accrues real domain computation (cart totals
  with discounts/tax, eligibility rules, a multi-step calculation), extract it into
  a **pure use-case / domain function** in the model layer (e.g.
  `models/pricing.ts → computeCartTotal(...)`) and have the VM call it. The split
  test: arithmetic/rules of the *domain* → model use-case (pure, unit-tested
  input→output); wiring those results into screen state → the VM. This keeps the VM
  a thin coordinator and the rules reusable and framework-free.
- **Cross-screen / cross-VM communication.** VMs never import each other. Share
  through the layer that owns the data: shared **client state** (a store slice) for
  app state, the **server-state cache** (an invalidated/refetched query) for server
  data, or **navigation params** for a one-shot hand-off. Reach for an event
  emitter only for fire-and-forget signals, behind a `shared/` facade.
- **Error handling, end-to-end.** `service` classifies transport/native failures
  into a domain `AppError` (a small union — see [`triad-example.md`](triad-example.md));
  the query/neutral hook surfaces it as `error`; the **VM** maps it to a ready
  message (`to*Message`) and `status: 'error'` (or a soft `banner` on `ready` when
  data is still shown); the **View** only renders it. Render-time exceptions are a
  separate axis — an **error boundary** registered at the **navigator/`app/`
  level** (wrapping the route tree) or as a dedicated `shared/components` wrapper,
  **not** as inline JSX inside a Screen (which would break the "Screen is wiring
  only" rule). Crash and analytics reporting fire from the VM or a thin service,
  never the View.
- **Derived state is computed, not stored.** Compute it in a `useMemo`/memoized
  selector (in the VM or the store layer) — never duplicate server/client state
  into another field that can drift.

---

## 2. Conformance checklist (the "keep it faithful" core)

Run these and report findings with `file:line` + severity. Adapt greps to the
language/framework.

**Triad**
- [ ] Each Screen is wiring only (no state/JSX/styles/logic).
- [ ] Each ViewModel imports no `react-native` and renders no JSX (importing `react` itself — `useState`/`useMemo`/etc. — is fine) and exposes an explicit interface/contract — **preferably a discriminated-union `status`** (see [`triad-example.md`](triad-example.md) §6) with ready values (formatted strings, booleans for visibility) inside each variant; flat flags (`error: string|null`) only for trivial cases (simple forms, triad §10).
- [ ] Each View formats nothing (`toFixed`, date formatting, display-string templating) and decides no error/empty/loading state — the VM exposes a discriminated `status`/booleans and the View only branches on it (a `switch`/early-return), never computing the condition itself. A `status` `switch` ends on `default: assertNever(vm)` so a new variant fails to compile and never renders `undefined` at runtime (see [`triad-example.md`](triad-example.md) §7).
- [ ] Views don't import services/queries/stores/navigation directly.

**Mapping layers**
- [ ] Wire→domain mapping is pure, isolated, named consistently; derived values are domain rules in the model, not the mapper.
- [ ] Display formatting lives in one place; primitives are shared (no duplicated `toFixed`/currency/date logic).
- [ ] User input is parsed/validated at a boundary with safe fallbacks (if the app has forms).

**SOLID**
- [ ] **S** — one reason to change per layer; teardown that spans concerns (e.g. logout = clear state + cache + navigate) lives in a coordinator, not the store.
- [ ] **O** — adding a screen/feature doesn't edit a central registry; clients/factories are extended, not modified.
- [ ] **L** — *(substitutability-by-contract: LSP read through the screen's VM interface, not class subtyping)* any ViewModel honoring a screen's contract is substitutable for the real one: a View behaves correctly with a fake VM that satisfies the interface (no hidden assumptions beyond the contract), which is exactly what makes the fake-VM test valid.
- [ ] **I** — consumers subscribe to the minimal store slice (selectors, not whole-store); public surfaces expose only what's needed.
- [ ] **D** — ViewModels depend on abstractions (navigation facade, services-behind-queries) and on **intent-named pure functions** (transformers/formatters/parsers). These pure functions form a stable input→output contract and are swappable without breaking callers, so depending on them directly is safe (DIP is about the *direction of dependency on a stable contract*, not about side-effect-freeness per se). The VM never depends on **concretions** (the HTTP client or the router directly) nor on unexported internal utilities — those are the DIP side-steps to catch in review.

**Structure & reuse**
- [ ] Layers present match the feature's responsibility — a *missing* `queries`/`mutations`/`services`/`models`/`screens` folder is fine (capability features have no screen; thin features no service/query). Do NOT flag absence against a fixed template.
- [ ] Reusable logic is a hook at the right tier (feature `hooks/` → shared `hooks/`); UI/interaction hooks hold no data; the ViewModel is the per-screen hook.
- [ ] Feature composition goes through public components/hooks; the cross-feature graph is acyclic and directional.

**Hygiene**
- [ ] Server state lives in the server-state layer (the query cache — which for RTK Query/Apollo physically sits in the store/client *by design*, and that's fine), not hand-duplicated into a client store slice; client state derived (not duplicated) where possible.
- [ ] User-facing copy and route strings are centralized (one place to change / i18n seam); the domain→string mapping lives in `formatters/` (which already turn domain → display strings), never in Views. **The locale reaches the formatter through the ViewModel, not a global** — see the i18n note below.
- [ ] Long lists are virtualized with stable keys and memoized rows — never a `.map()` inside a `ScrollView`. Use the platform's virtualized list (RN core `FlatList`, or an alternative such as `FlashList`): a stable `keyExtractor` (no index keys), a memoized `renderItem` + `memo()`'d rows, and — for `FlatList` — tuned windowing (`initialNumToRender`/`windowSize`/`removeClippedSubviews`, plus `getItemLayout` for fixed-height rows). Any specific list library's own perf props follow *its current docs* (they change across majors); the version-proof rule is the principle — virtualize, stable keys, memoized rows.
- [ ] Remote/large images are sized and cached deliberately — explicit `width`/`height` (no unbounded intrinsic size), a sensible `resizeMode`, and a caching layer when scroll performance demands it. The *principle* is the rule; the library is just an instance (RN's core `Image` caches modestly; a dedicated caching-capable component — e.g. `expo-image` — is an **option, not a requirement**, picked like any other entry in [`stack-choices.md`](stack-choices.md)). Unbounded/uncached images are a common RN memory + scroll-jank source. (Image choice is a View concern; the source URL is a ready value the VM hands down.)
- [ ] VM outputs passed to memoized Views have **stable references** (`useCallback`/`useMemo`); recreating handlers/objects each render defeats `memo()` and the virtualization gains above.
- [ ] Derived/time-based display is computed at load (in a transformer/formatter or on mount), **not** via a `setInterval` re-render — *unless* the display is genuinely live (a countdown, ticking relative time), in which case the ticking lives in a dedicated **UI hook** (which holds no data), never scattered in a View or VM body.
- [ ] Views carry accessibility metadata — a11y is a View concern, like layout: semantic **roles/labels** (`accessibilityRole`, `accessibilityLabel`/`accessibilityHint`, `accessibilityState` for selected/disabled/busy); **touch targets ≥ 44×44pt on iOS / 48×48dp on Android** (`hitSlop` where needed); a sensible **screen-reader reading order** (group related nodes with `accessible`, set `accessibilityElementsHidden`/`importantForAccessibility` on decorative layers); **respects OS text scaling** (don't blanket-disable `allowFontScaling`; let layouts reflow) and **reduce-motion** (`AccessibilityInfo.isReduceMotionEnabled` gates non-essential animation). Accessibility **copy** is user-facing text → it follows the same centralization/i18n rule (it comes from the VM/constants, never an inline literal — see the i18n note and triad §7). Whether an element is *announced* may be a VM decision (a ready `accessibilityLabel` string on the view-item); how it's *rendered* is the View's.
- [ ] Render-time failures are contained by an **error boundary** registered at the navigator/`app/` level (or a dedicated `shared/components` wrapper), with a typed fallback — not inline JSX in a Screen.
- [ ] Co-located tests per layer; pure layers (transformers/formatters/parsers/rules) tested input→output; ViewModels tested via their contract; Views with a fake VM.
- [ ] No `any`/dead code/leftover logs; strict typing on.

> **Note — i18n and *pure* formatters.** A pure formatter can't read "the current
> locale" reactively, so localization can't simply "live in a formatter" with no
> plumbing. The locale enters the **ViewModel** through a neutral hook (`useLocale()`
> or a thin `useTranslation()` wrapper); the VM then either passes the resolved
> `t`/locale into the formatter as an argument (`toErrorMessage(error, t)`) or hands
> the View an already-translated `message`. The reactive re-render on a language
> switch is owned by the VM's hook — **never** by a global i18n singleton read inside
> a "pure" formatter, which reintroduces the hidden dependency this contract bans. The
> formatter stays pure (locale in → string out); only *where the locale comes from*
> is specified here. **This holds for *every* locale-dependent formatter, not just
> `to*Message`:** `formatPrice`/`formatDate`/`toProductItem` take the resolved
> `locale` (e.g. `Intl.NumberFormat(locale, …)`) as an argument the same way. The
> worked slices in [`triad-example.md`](triad-example.md) call them with no locale
> argument purely to stay short — a localized app threads `locale`/`t` from the VM
> into each.

---

## 3. The scaling ladder

```
   complexity & team size →
   Rung 1 SCREEN-BASED  →  Rung 2 FEATURE-BASED  →  Rung 3a MODULAR MONOLITH  →  Rung 3b MICRO-FRONTEND
   group by screen         vertical slices,          features = packages with     features = independently
   one shared src/         public API per feature     enforced build boundaries     built & deployed units
```

**Decision tree — climb only when the pain is real:**

- **Stay screen-based** for a small–medium app, one small team, shared models/services. *Outgrown when:* flat `screens/`/`services/`/`queries/` get crowded; unrelated screens share a neighborhood; people step on each other in one shared tree.
- **Go feature-based** when there are several distinct areas and a few teams, and you want each area understandable/ownable in isolation. *Outgrown when:* features must be **built, released, or owned** independently.
- **Go modular monolith** when you need enforced isolation but still one build/deploy — features become packages; cross-feature imports blocked by tooling. *Usually the right next step* (most of the benefit, far less cost than MFE).
- **Go micro-frontend** only when independent **deployment** (not just development) is a genuine requirement — separate teams, separate release pipelines, runtime composition. Real overhead (versioning, contracts, shared-deps, composition).

> **The ladder is reversible.** If a rung's isolation isn't paying for its cost,
> climb *down*: merge thin features back together, or collapse a micro-frontend
> into the modular monolith (re-internalize the package, drop the versioned
> contract). Because the triad is identical on every rung, demotion is the same
> mechanical reorg as promotion, run backwards. Don't treat a rung as permanent.

> The triad still lives inside every rung. Only the wall around the units gets taller.

---

## 4. Migration playbooks (mechanical moves per step)

### Recovering fidelity in a drifted codebase (any rung)

The climbs below *add* a rung. This one is orthogonal: the layers exist but are
**wrong** (Views format, VMs import `react-native`/the router, services scattered,
state living in components). Restore fidelity incrementally — don't rewrite.

1. Run the conformance checklist (§2) on one representative screen; record the top 3
   violations with `file:line`.
2. Move each violation to its owning layer — a *move*, not a redesign:
   - **View formats** (`toFixed`/date/template) → create a `formatters/` fn; the VM
     passes the ready string.
   - **VM imports `react-native`/the router** → route UI through props; route
     navigation through a `navigation/` facade (intent-named methods).
   - **Services scattered / `fetch`/axios in a VM or View** → cage I/O in `services/`;
     expose a neutral query hook the VM depends on.
   - **State in a component** → lift it into the ViewModel hook.
3. Commit **per layer**, keeping tests green between commits (small, reversible steps).
4. Re-run the checklist on the next screen. Fidelity is restored screen-by-screen, not
   in a big-bang refactor — and only *then* consider climbing a rung if scale demands it.

### Screen-based → Feature-based
1. Create `features/<name>/` per area; move that area's screens + its owned layers (models, transformers/formatters/parsers, services, queries, components, navigation) into it. **Co-located → by-type:** at Rung 1 the triad sits together in `screens/<S>/` (`<S>Screen.tsx`, `<S>View.tsx`, `use<S>ViewModel.ts`); when you promote the feature, spread them into `viewmodels/`, `views/`, `screens/` — a *move*, not a rewrite. If the `<Screen>VM` contract was co-located in the hook file, give it its own file here (see [`conventions.md`](conventions.md)). Don't pre-create these by-type folders at Rung 1.
2. Give each feature a **public API** (a barrel `index.ts`): export only screens + the types/components others may use; keep internals private.
3. **Intra-feature imports relative**; cross-feature/app imports go through the feature's public root only.
4. Keep genuinely cross-cutting code in `shared/`; **`shared/` must never import a feature**.
5. **Enforce the boundary with lint** (e.g. `no-restricted-imports`): forbid deep cross-feature imports + any `features/*` import from `shared/`.
6. Break cross-cutting cycles by **dependency inversion**: if a shared layer (e.g. the HTTP client's auth interceptor) needs a feature, define a *port* in shared and have the feature implement it and register at boot.
7. Split app-shell vs feature-owned **navigation**: shell transitions (login/home) in shared; deep routes owned by each feature.
8. Compose features through public components/hooks (never deep imports); keep the dependency graph **acyclic and directional**.

### Feature-based → Modular monolith
1. Promote each `features/<name>/` to a **package** (workspace) with its own entry point (its public API becomes the package's exports).
2. Move `shared/` to a `shared`/`core` package.
3. Enforce boundaries at **build time** (package boundaries / project references / a boundaries lint plugin): a feature package may depend only on `shared` + its own internals; cross-feature only via published entry points.
4. One build, one deploy — but modules are genuinely decoupled and independently testable.
> **RN-specific (the hardest mechanical step):** pick the monorepo tooling
> (npm/yarn/pnpm workspaces, or Nx/Turborepo) and configure **Metro** for it —
> `watchFolders` + `nodeModulesPaths` so Metro resolves hoisted/symlinked packages,
> and **one** `react`/`react-native` instance (dedupe; never two copies). Keep the
> TS path aliases and the Metro resolver in sync.

### Modular monolith → Micro-frontend
1. Give each feature/package an **independent build + deploy** pipeline.
2. Define **versioned contracts** between units — a shared **types package** (or runtime-validated payloads at the seam) *is* the contract; manage shared dependencies explicitly (singletons, version ranges).
3. **Compose at runtime** (in RN: Re.Pack's Module Federation, or a native brownfield / mini-app host); host shell loads units.
4. Only worth it when independent *deployment* is required; otherwise stay a modular monolith.
> **RN landmines:** `react`/`react-native` (and any lib holding native state — the
> navigation/gesture/reanimated libs) **must be singletons** — a second copy
> crashes at runtime; share them as host-provided externals. Watch version skew
> across units, Hermes bytecode compatibility, and how remote units interact with
> OTA (EAS Update, or the community CodePush fork — Microsoft's App Center CodePush
> was retired in 2025) — a federated remote and an OTA bundle are two update
> channels that must agree.

---

## 5. Output formats

- **Conformance review:** Strengths (brief) · Issues (Critical/Important/Minor, each `file:line` + the rule + a concrete fix) · Verdict (Faithful / Minor deviations / Material violations).
- **Scaling plan:** detected rung → recommended rung (with the specific "outgrown when" signals that justify it, or a recommendation to stay) → a **phased migration plan** (each phase independently shippable + verifiable: typecheck, lint, tests, boundary checks green).

---

## 6. Where does this go? (quick placement)

| You're adding… | It goes in… |
|---|---|
| A new screen | a Screen+View+ViewModel triad (`screens/` or `features/<f>/screens/`) |
| Visual layout / styling | the **View** |
| Screen state / an event handler / "what to show" | the **ViewModel** |
| Domain computation (totals, eligibility) | a **use-case** in the model — the VM *calls* it |
| Reusable UI/interaction logic | a **hook** (`features/<f>/hooks/` → `shared/hooks/`) |
| API response → domain | a **transformer** (`transform*`) |
| Domain → display string | a **formatter** (`to*`) |
| User input → value | a **parser** (`parse*`) |
| A new HTTP call | a **service** |
| Caching / pagination / retry | a **query/mutation** |
| Global client state | a **store** (teardown spanning concerns → a coordinator, not the store) |
| A navigation target | the navigation facade (shell) or the feature's `navigation.ts` (deep route) |
| User-facing text / routes | centralized constants |
| Something one feature needs from another | export it from that feature's public API; import the feature root |

If a change doesn't fit one row, it's probably two changes in two layers — split it.

---

## 7. Keeping it faithful over time (governance)

Architecture decays without enforcement. For multi-team / multi-year projects:
- **Automate the gates.** Run typecheck + lint + tests + the boundary greps/lint in
  CI on every PR, and in a pre-commit/pre-push hook — not just by hand. A boundary
  the tooling enforces is the only one that survives turnover. A minimal CI job (swap
  the runner for the project's — bun/npx/yarn/pnpm):

  ```yaml
  # .github/workflows/architecture-gates.yml
  on: [pull_request]
  jobs:
    gates:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - uses: actions/setup-node@v4
          with: { node-version: 20, cache: npm }
        - run: npm ci
        - run: npx tsc --noEmit          # type gate (0 errors)
        - run: npx eslint .              # lint + the no-restricted-imports boundary rules (§4 playbook / worked-examples §2)
        - run: npm test -- --ci          # the behavior gate
        # feature-based and up: fail the build on a deep cross-feature import.
        # test-utils is opted out, to match the ESLint rule + the §1 grep in worked-examples.
        - run: "! grep -rEn '@/features/[a-z0-9-]+/[a-zA-Z]' src --include='*.ts' --include='*.tsx' | grep -v 'src/shared/test-utils/'"
  ```
  This minimal job gates the deep-import boundary only; the `shared/ → feature` and
  alias-to-self checks (worked-examples §1) ride along on the `eslint .` step above.
- **Assign ownership.** `CODEOWNERS` per feature/package, so the owning team reviews
  changes that cross their public API.
- **Record decisions.** Capture architectural choices (a rung climb, a new port, a
  stack swap) as short ADRs, so the *why* outlives the people who decided it.
