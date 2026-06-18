# Cheat sheet — the 60-second orientation

The one-screen digest of the whole contract. Stack-agnostic and structure-agnostic:
nothing here names a library or a rung — it holds on every RN app. Each row points at
the deep reference that expands it. Skim this first; read the named file when you need
the *why*.

---

## The triad + composition root (never changes — any stack, any rung)

| Layer | One-line job | Must NOT | Deep ref |
|---|---|---|---|
| **Model** | what the data *is* + domain rules (`isTopRated`, `computeCartTotal`) | import UI / HTTP / framework | mvvm-and-scaling [section 1](mvvm-and-scaling.md#1-layer-responsibilities-the-contract) |
| **Transformer** `transform*` | wire DTO ↔ domain (rename, fallbacks, representation derivations) | format; do I/O; compute domain rules | triad [section 3](triad-example.md#3-transformer--wire--domain-pure) |
| **Formatter** `to*` | domain → display string / view-item; `to*Message` for errors | mutate; do I/O; compute domain rules | triad [section 4](triad-example.md#4-formatter--domain--view-item-pure-display-ready) |
| **Parser** `parse*` / `validate*` | user input → typed value / typed *fault* | produce the user-facing message | triad [section 10](triad-advanced.md#10-controlled-inputs-forms--the-view-stays-dumb) |
| **Service** `<verb><Noun>` | the **only** I/O (Input/Output) — network **and** native/device; classify failures → `AppError` | format; cache; be called from a View | triad [section 2](triad-example.md#2-service--the-only-layer-that-touches-io-classifies-errors) |
| **Query/Mutation** | caching, pagination, retry, invalidation, optimistic update | UI; formatting; be consumed outside a VM | triad [section 5](triad-example.md#5-neutral-feature-hook--wraps-the-server-state-lib-so-the-vm-never-sees-it), [section 17](triad-advanced.md#17-mutations--the-write-path-behind-a-neutral-hook-invalidation--optimistic-update) |
| **Store** | global **client** state; pure container | navigate; clear caches; hold server state | worked [section 8b](worked-examples.md#8-the-boundaries-in-code-this-stack-expo-router--zustand--tanstack--axios) |
| **Persistence** | durable-storage adapter (secure store / KV) | be imported by a VM/View | conventions glossary |
| **Coordinator** | one cross-concern action (logout = clear state + cache + navigate) | hold state; format; be imported by a View | worked [section 8c](worked-examples.md#8-the-boundaries-in-code-this-stack-expo-router--zustand--tanstack--axios) |
| **ViewModel** | screen state + behavior + "what to show" (a discriminated `status`) | import `react-native`/JSX; touch HTTP/router; format inline | triad [section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union) |
| **View** | render the branch the VM resolved; forward events; may consume a **pure UI hook** (`use<Behavior>`, holds no data) directly | format; decide loading/error/empty; call service/store/query/nav | triad [section 7](triad-example.md#7-view--passive-only-branches-on-status-formats-nothing), [section 21](triad-crosscutting.md#21-animations--gestures--a-ui-hook-that-holds-no-data-never-the-viewmodel) |
| **Screen** | wiring: build the VM, pass its output into the View as props | hold state / JSX / styles / logic | triad [section 8](triad-example.md#8-screen--the-per-screen-composition-root) |

**Data flows VM → (props) → View; events flow View → (callbacks) → VM.** The View's
only compile-time dependency on the VM is the **contract module** (`<Screen>VM`,
imported `type`-only by both). External libraries sit at the edges, behind abstractions.

> The one rule everything follows: **each piece of code has exactly one reason to
> change, and lives in the layer that owns that reason.** One edit forcing three layers
> = a responsibility leaked.

---

## Where does X go?

| You're adding… | It goes in… |
|---|---|
| A new screen | a Screen + View + ViewModel triad |
| Layout / styling / a11y (accessibility) metadata | the **View** |
| An independent visual block (own behavior / separate responsibility) | a **component** in `components/`; the **View orchestrates** the composition (conventions [View composition](conventions.md#view-composition--orchestrate-extract-by-cohesion)) |
| Screen state / a handler / "what to show" | the **ViewModel** |
| API response → domain | a **transformer** (`transform*`) |
| Domain → display string | a **formatter** (`to*`) |
| User input → value | a **parser** (`parse*`) |
| A new HTTP/native call | a **service** |
| Caching / pagination / retry / optimistic write | a **query/mutation** |
| Global client state | a **store** (cross-concern teardown → a **coordinator**) |
| A navigation target | the **navigation facade** (shell) or feature `navigation.ts` (deep route) |
| Domain computation (totals, eligibility) | a **use-case** in the model — the VM *calls* it |
| View-local presentation behavior (animation, gesture, keyboard, scroll, toggle, dimensions) | a **UI hook** (`use<Behavior>`, holds no data) the **View consumes directly** — not the ViewModel (triad [section 21](triad-crosscutting.md#21-animations--gestures--a-ui-hook-that-holds-no-data-never-the-viewmodel)) |
| Reusable UI/interaction logic | a **hook** (`features/<f>/hooks/` → `shared/hooks/`) |
| User-facing text / route strings | centralized constants / i18n |
| Something one feature needs from another | export it from that feature's public API; import the feature root |

If a change doesn't fit one row, it's two changes in two layers — split it.

> This is the canonical placement table; the contract doc
> ([`mvvm-and-scaling.md`](mvvm-and-scaling.md#6-where-does-this-go-quick-placement))
> points here rather than repeating it.

---

## Red flags (stack-neutral — the lib in parens is just one instance)

- View **formatting** (`.toFixed`, dates, `` `@${u}` ``) or **deciding** error/empty/loading → move to a formatter / a discriminated `status`.
- View **concentrating many independent visual blocks** / inline render states in one file → extract cohesive blocks to `components/`, let the View orchestrate (by cohesion, not line count).
- A **domain entity / DTO / internal structure reaching the View** (a raw `Product`/`ProductDTO` in the contract) → expose render-ready view-items + actions only.
- ViewModel importing **`react-native`** / rendering JSX, or touching the **router** directly → rendering to the View, navigation behind the facade.
- **HTTP client** (`axios`/`fetch`/`ky`) outside `services/` → confine transport.
- **Native API** (`expo-camera`, `expo-location`) outside its adapter → confine like HTTP.
- A lib's **result shape** in a VM (`hasNextPage`, `isFetchingNextPage`) → a feature-defined neutral hook.
- **Whole-store subscription** (`useStore()`) when a selector exists → subscribe to the slice (ISP).
- **Cross-concern teardown** inside a store → a coordinator.
- **Server state hand-duplicated** into a client store → keep it in queries/mutations.
- Feature-based+: **deep cross-feature import** or **`shared/` importing a feature** → break the boundary.

Concrete greps: worked-examples [section 1](worked-examples.md#1-conformance-greps-run-from-the-project-root) (Expo + TanStack) and [section 10](worked-examples.md#10-the-same-recipes-on-a-contrasting-stack-bare-rn--react-navigation--redux-toolkit--fetch) (bare RN + RN + Redux + fetch).
Each violation worked end-to-end (symptom → rule broken → fix): worked-examples [section 11](worked-examples.md#11-common-adoption-mistakes-and-the-boundary-each-one-breaks); the
full "god component" refactored layer-by-layer: triad [section 13](triad-advanced.md#13-anti-pattern--the-god-component-that-does-everything-refactored).

---

## Scale only when the pain is real

```
Rung 1 SCREEN-BASED → Rung 2 FEATURE-BASED → Rung 3a MODULAR MONOLITH → Rung 3b MICRO-FRONTEND
```

- **Stay** until the current shape genuinely hurts; the ladder is **reversible**.
- **Climb to feature-based** when several areas/teams need isolated ownership.
- **Modular monolith** when you need enforced isolation but one build (usually the right next step).
- **Micro-frontend** only when independent **deployment** is a real requirement.

The triad is identical on every rung; only *how files are grouped* changes. Decision
tree + migration playbooks: mvvm-and-scaling [sections 3–4](mvvm-and-scaling.md#3-the-scaling-ladder).

---

## Build the fewest layers that earn their keep (adoption ladder)

1. Tiny app → Model + View + ViewModel + Screen, one service, one formatter (triad [section 0](triad-example.md#0-quickstart--the-smallest-faithful-slice-rung-1-40-lines) quickstart).
2. + caching/refetch/pagination → `queries/` + `mutations/` (or a neutral data hook).
3. + shared client state → `stores/`; a cross-concern teardown → a `coordinator`.
4. + forms → `parsers/`.
5. Several areas/teams → climb to feature-based.

A **missing** `queries/`/`mutations/`/`services/`/`models/`/`screens/` folder is **not a
defect** — judge a feature by its responsibility. Over-building is flagged as readily as
under-structuring. Full layer vocabulary = the toolbox, not a day-one checklist.

---

## The gates (run on every change — CI + pre-commit)

`tsc --noEmit` = 0 · the project's lint = 0 · tests green · (feature-based+) boundary
greps/lint clean. Use the project's own runner (bun → `bunx`; else `npx`/`yarn`/`pnpm`).
A boundary the tooling enforces is the only one that survives turnover (mvvm-and-scaling [section 7](mvvm-and-scaling.md#7-keeping-it-faithful-over-time-governance)).

---

## Naming (the prefix means exactly one thing on every rung)

`use<Screen>ViewModel` · `use<Section>ViewModel` (sub-VM, composed by the screen VM) ·
`<Screen>VM` (contract type, own file) · `<Screen>View` ·
`<Screen>Screen` · `transform*` (wire↔domain) · `to*` (domain→view) · `parse*`
(input→value) · `validate*` (input→fault) · `use<Thing>Data` (neutral data hook, in
`queries/`) · `use<Behavior>` (UI hook, holds no data) · `<Name>` (extracted component,
PascalCase, no `use`) · folders plural/lowercase ·
co-located `*.test.ts(x)`. Full table: conventions.md.

`use*` means **hook** and nothing else — contracts/types/interfaces (`<Screen>VM`,
`ProductItem`, DTOs) never take it. No reflexive `types.ts`: co-locate a type with its
owner, extract only when shared across files or it dominates the file.

---

## The deep references

- **mvvm-and-scaling.md** — the layer contract, conformance checklist, **SOLID explained**, scaling ladder, migration playbooks, governance.
- **triad-example.md** — the triad in code, **core slice** (quickstart [section 0](triad-example.md#0-quickstart--the-smallest-faithful-slice-rung-1-40-lines) → Model→Service→Transformer→Formatter→Query→VM→View→Screen→Tests, sections 0–9).
- **triad-advanced.md** — the harder cases (sections 10–18): forms, Open/Closed, the no-server-state path, the god-component refactor, route params, the same triad on MobX/RTK Query/Redux [section 15](triad-advanced.md#15-the-same-triad-on-another-stack--a-mobx-class-vm-an-rtk-query-swap-and-a-redux-client-state-vm), error boundary, mutations, pagination.
- **triad-crosscutting.md** — cross-cutting concerns (sections 19–23): i18n, accessibility, animations, Suspense, plus a [referenced-helpers appendix](triad-crosscutting.md#23-referenced-helpers--primitives-assumed-not-re-implemented-here).
- **conventions.md** — glossary, naming, canonical folder trees, VM-state modeling, the adoption ladder, JS-only degradation.
- **stack-choices.md** — the library menu per concern, where each plugs into the boundaries, how swappable each is.
- **worked-examples.md** — concrete recipes on Expo + TanStack ([sections 1–9](worked-examples.md#1-conformance-greps-run-from-the-project-root)) **and** a contrasting stack — bare RN + React Navigation + Redux Toolkit + fetch ([section 10](worked-examples.md#10-the-same-recipes-on-a-contrasting-stack-bare-rn--react-navigation--redux-toolkit--fetch)).
- **integration-recipes.md** — the previously-deferred concerns worked in code: observability, real-time, offline-first/sync, GraphQL, deep linking, security — each behind one layer's neutral contract.
