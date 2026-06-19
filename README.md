# react-native-mvvm-guardian

> MVVM Guardian for React Native, including the rules necessary to keep MVVM healthy as the application scales.

A Claude Code **plugin** that keeps a **React Native** app faithful to **MVVM +
SOLID + best practices + conventions**, helps you **choose your stack** without
breaking the architecture, and — when the app outgrows its shape — knows **how to
scale it** up a deliberate ladder:

> **New to the jargon?** **MVVM** = Model–View–ViewModel (the three-layer split this
> plugin enforces). **SOLID** = five object-oriented design principles — Single
> Responsibility, Open–Closed, Liskov Substitution, Interface Segregation, and
> Dependency Inversion. You don't need to know them going in: every term is spelled
> out and shown with examples in the references (the glossary in
> [`conventions.md`](skills/rn-mvvm-guardian/references/conventions.md#glossary), and
> SOLID worked principle-by-principle in
> [`mvvm-and-scaling.md`](skills/rn-mvvm-guardian/references/mvvm-and-scaling.md#solid--the-five-principles-in-plain-terms-and-in-mvvm)).

```
screen-based → feature-based → modular monolith → micro-frontend
```

It ships **one pair** — a skill and an agent, both named `rn-mvvm-guardian`:

| Component | What it does |
|---|---|
| **`rn-mvvm-guardian` skill** | Loads the MVVM contract, the conformance checklist, the scaling ladder + migration playbooks, and the guide to **where each library plugs into the boundaries**. Stack-agnostic and structure-agnostic. |
| **`rn-mvvm-guardian` agent** | Audits a codebase for MVVM/SOLID faithfulness, advises which lib belongs where (and whether you can swap it), and drafts/executes scaling migrations — running the real gates and citing evidence. |

It works for **any** RN app — **whatever your stack** (React Navigation *or*
expo-router, with *or* without TanStack Query/SWR, fetch *or* axios, Zustand/Redux/
MobX/Context/Jotai) and **whatever your structure** (screen-based through micro-frontend). The
MVVM triad and every layer's responsibility never change; only *how files are
grouped* (the rung) and *which library sits inside a layer* do. A concrete **Expo +
expo-router + TanStack Query + Zustand + axios** instantiation is included as a
copy-and-adapt **worked example** — not a separate, competing skill.

The north star is constant: **guide toward MVVM, SOLID, best practices, and
conventions.** You keep the power to choose your libraries and your structure; the
skill keeps each choice behind its boundary so the app stays faithful and the
choice stays reversible.

> **New here?** Skim the [cheatsheet](skills/rn-mvvm-guardian/references/cheatsheet.md)
> for a 60-second orientation, then copy the
> [section 0 quickstart](skills/rn-mvvm-guardian/references/triad-example.md#0-quickstart--the-smallest-faithful-slice-rung-1-40-lines) (~40 lines —
> the smallest faithful triad) and add layers only when a distinct reason to change
> appears. Everything else is reference you reach for on demand.

## Install

This repo self-hosts as a plugin marketplace (`.claude-plugin/marketplace.json`).
In Claude Code:

```
/plugin marketplace add alexandrecoura96/react-native-mvvm-guardian
/plugin install rn-mvvm-guardian@react-native-mvvm-guardian
```

(Update the owner if you fork/transfer the repo.) Then:
- **Skill:** `/rn-mvvm-guardian:rn-mvvm-guardian` — Claude also auto-invokes it based on its `description`.
- **Agent:** appears in `/agents` (`rn-mvvm-guardian`).

To try it before publishing, add a local path instead:
`/plugin marketplace add ./react-native-mvvm-guardian`.

## What's inside

- `skills/rn-mvvm-guardian/SKILL.md` — the stack- and structure-agnostic skill (triad,
  boundaries, activation flow, stack-neutral red flags).
- `skills/rn-mvvm-guardian/references/`:
  - `cheatsheet.md` — the one-screen digest (triad table, where-does-X-go, red flags,
    the ladder, the gates, naming), each row pointing at the deep reference. Start here.
  - `mvvm-and-scaling.md` — the layer contract, conformance checklist, scaling
    ladder, migration playbooks, and governance.
  - `triad-example.md` — the triad in code, **core slice** (sections 0–9): a full
    Model→…→View→Screen slice, the ViewModel contract as a discriminated union,
    and tests.
  - `triad-advanced.md` — the **harder cases** (sections 10–18): forms, Open/Closed via
    a registry, the no-server-state path, the god-component refactor, typed route params,
    mutations (optimistic + rollback), an error boundary, and the same triad on
    MobX / RTK Query / Redux.
  - `triad-crosscutting.md` — the **cross-cutting concerns** (sections 19–23): i18n,
    accessibility, animations, Suspense, plus a referenced-helpers appendix listing every
    assumed primitive (`formatPrice`, `Spinner`, `useLoginMutation`, …) with its signature.
  - `conventions.md` — glossary, naming rules, canonical folder trees, VM-state
    modeling, the adoption ladder, and what's out of scope (so far).
  - `stack-choices.md` — the library menu per concern, where each plugs into the
    boundaries, and how swappable each is.
  - `worked-examples.md` — a concrete Expo + TanStack instantiation: conformance
    greps, the feature-boundary ESLint recipe, the `AuthBridge` inversion,
    server-/client-state specifics, and the worked screen→feature migration —
    **plus section 10, the same recipes re-derived on a contrasting stack** (bare RN +
    React Navigation + Redux Toolkit + fetch) so the patterns are shown changing
    *imports only*. Keep the pattern, swap the specifics for your libs.
  - `integration-recipes.md` — the previously-deferred concerns worked in code:
    observability, real-time, offline-first/sync, GraphQL, deep linking, and security —
    each kept behind one layer's neutral contract (so a library swap stays one-layer).
- `agents/rn-mvvm-guardian.md` — the audit + stack-choice advice + scaling agent.

## Provenance

The contract is distilled from production React Native MVVM practice — the
Model/View/ViewModel triad adapted to React's unidirectional data flow, SOLID, and
the feature-boundary discipline. It is **self-contained in this repo**: the cheatsheet
(index) plus the deep references above are the long-form source for the contracts, the boundary lint, the
transformer/formatter/parser taxonomy, the triad in code, and the worked
screen→feature migration.

## Status

v0.1.0 — first pass. One stack- and structure-agnostic pair (`rn-mvvm-guardian`), with a
concrete Expo + TanStack worked example folded in. See `skills/rn-mvvm-guardian/SKILL.md`
for the activation flow.
