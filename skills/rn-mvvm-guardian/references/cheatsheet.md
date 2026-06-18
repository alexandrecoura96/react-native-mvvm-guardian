# Cheat sheet — the 60-second orientation

The one-screen digest of the whole contract. Stack-agnostic and structure-agnostic:
nothing here names a library or a rung — it holds on every RN app. Each row points at
the deep reference that expands it. Skim this first; read the named file when you need
the *why*.

---

## The triad + composition root (never changes — any stack, any rung)

| Layer | One-line job | Must NOT | Deep ref |
|---|---|---|---|
| **Model** | what the data *is* + domain rules (`isTopRated`, `computeCartTotal`) | import UI / HTTP / framework | mvvm-and-scaling §1 |
| **Transformer** `transform*` | wire DTO ↔ domain (rename, fallbacks, representation derivations) | format; do I/O; compute domain rules | triad §3 |
| **Formatter** `to*` | domain → display string / view-item; `to*Message` for errors | mutate; do I/O; compute domain rules | triad §4 |
| **Parser** `parse*` / `validate*` | user input → typed value / typed *fault* | produce the user-facing message | triad §10 |
| **Service** `<verb><Noun>` | the **only** I/O — network **and** native/device; classify failures → `AppError` | format; cache; be called from a View | triad §2 |
| **Query/Mutation** | caching, pagination, retry, invalidation, optimistic update | UI; formatting; be consumed outside a VM | triad §5, §17 |
| **Store** | global **client** state; pure container | navigate; clear caches; hold server state | worked §8b |
| **Persistence** | durable-storage adapter (secure store / KV) | be imported by a VM/View | conventions glossary |
| **Coordinator** | one cross-concern action (logout = clear state + cache + navigate) | hold state; format; be imported by a View | worked §8c |
| **ViewModel** | screen state + behavior + "what to show" (a discriminated `status`) | import `react-native`/JSX; touch HTTP/router; format inline | triad §6 |
| **View** | render the branch the VM resolved; forward events; may consume a **pure UI hook** (`use<Behavior>`, holds no data) directly | format; decide loading/error/empty; call service/store/query/nav | triad §7, §21 |
| **Screen** | wiring: build the VM, pass its output into the View as props | hold state / JSX / styles / logic | triad §8 |

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
| Layout / styling / a11y metadata | the **View** |
| Screen state / a handler / "what to show" | the **ViewModel** |
| API response → domain | a **transformer** (`transform*`) |
| Domain → display string | a **formatter** (`to*`) |
| User input → value | a **parser** (`parse*`) |
| A new HTTP/native call | a **service** |
| Caching / pagination / retry / optimistic write | a **query/mutation** |
| Global client state | a **store** (cross-concern teardown → a **coordinator**) |
| A navigation target | the **navigation facade** (shell) or feature `navigation.ts` (deep route) |
| Domain computation (totals, eligibility) | a **use-case** in the model — the VM *calls* it |
| View-local presentation behavior (animation, gesture, keyboard, scroll, toggle, dimensions) | a **UI hook** (`use<Behavior>`, holds no data) the **View consumes directly** — not the ViewModel (triad §21) |
| User-facing text / route strings | centralized constants / i18n |

If a change doesn't fit one row, it's two changes in two layers — split it.

---

## Red flags (stack-neutral — the lib in parens is just one instance)

- View **formatting** (`.toFixed`, dates, `` `@${u}` ``) or **deciding** error/empty/loading → move to a formatter / a discriminated `status`.
- ViewModel importing **`react-native`** / rendering JSX, or touching the **router** directly → rendering to the View, navigation behind the facade.
- **HTTP client** (`axios`/`fetch`/`ky`) outside `services/` → confine transport.
- **Native API** (`expo-camera`, `expo-location`) outside its adapter → confine like HTTP.
- A lib's **result shape** in a VM (`hasNextPage`, `isFetchingNextPage`) → a feature-defined neutral hook.
- **Whole-store subscription** (`useStore()`) when a selector exists → subscribe to the slice (ISP).
- **Cross-concern teardown** inside a store → a coordinator.
- **Server state hand-duplicated** into a client store → keep it in queries/mutations.
- Feature-based+: **deep cross-feature import** or **`shared/` importing a feature** → break the boundary.

Concrete greps: worked-examples §1 (Expo + TanStack) and §10 (bare RN + RN + Redux + fetch).
Each violation worked end-to-end (symptom → rule broken → fix): worked-examples §11; the
full "god component" refactored layer-by-layer: triad §13.

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
tree + migration playbooks: mvvm-and-scaling §3–§4.

---

## Build the fewest layers that earn their keep (adoption ladder)

1. Tiny app → Model + View + ViewModel + Screen, one service, one formatter (triad §0 quickstart).
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
A boundary the tooling enforces is the only one that survives turnover (mvvm-and-scaling §7).

---

## Naming (the prefix means exactly one thing on every rung)

`use<Screen>ViewModel` · `<Screen>VM` (contract type, own file) · `<Screen>View` ·
`<Screen>Screen` · `transform*` (wire↔domain) · `to*` (domain→view) · `parse*`
(input→value) · `validate*` (input→fault) · `use<Thing>Data` (neutral data hook, in
`queries/`) · `use<Behavior>` (UI hook, holds no data) · folders plural/lowercase ·
co-located `*.test.ts(x)`. Full table: conventions.md.

---

## The six deep references

- **mvvm-and-scaling.md** — the layer contract, conformance checklist, scaling ladder, migration playbooks, governance.
- **triad-example.md** — the triad in code (quickstart §0 → full slice → mutations, route params, error boundary, the same triad on MobX/RTK Query/Redux §15).
- **conventions.md** — glossary, naming, canonical folder trees, VM-state modeling, the adoption ladder, JS-only degradation.
- **stack-choices.md** — the library menu per concern, where each plugs into the boundaries, how swappable each is.
- **worked-examples.md** — concrete recipes on Expo + TanStack (§1–§9) **and** a contrasting stack — bare RN + React Navigation + Redux Toolkit + fetch (§10).
- **integration-recipes.md** — the previously-deferred concerns worked in code: observability, real-time, offline-first/sync, GraphQL, deep linking, security — each behind one layer's neutral contract.
