# A worked triad in code — cross-cutting concerns (sections 19–23)

> Continues [`triad-example.md`](triad-example.md) (the core slice, sections 0–9) and
> [`triad-advanced.md`](triad-advanced.md) (advanced cases, 10–18) — same example, same
> boundaries. These sections keep each cross-cutting concern on the right side of the triad:
> localization (i18n), accessibility, animations/gestures, Suspense — plus the
> referenced-helpers appendix listing every assumed primitive.

---

## 19. Localization (i18n) — the locale enters through the VM, formatters stay pure

The prose rule (the i18n note in [`mvvm-and-scaling.md`](mvvm-and-scaling.md) [section 2](mvvm-and-scaling.md#2-conformance-checklist-the-keep-it-faithful-core)) in
code: a pure formatter **cannot** read "the current locale" — so the locale is threaded
in *from the VM*, and the formatter takes it as an argument. The reactive re-render on
a language switch is owned by the VM's neutral `useTranslation()` hook, **never** by a
global singleton read inside a "pure" formatter. This is the single most-often-botched
boundary, so here it is end to end.

```ts
// shared/i18n/index.ts — the neutral hook the VM consumes; wraps whatever i18n lib you use
import { useTranslation as useI18n } from 'react-i18next'; // the ONLY place the lib is named
export function useTranslation(): { t: (key: string, vars?: Record<string, unknown>) => string; locale: string } {
  const { t, i18n } = useI18n();
  return { t, locale: i18n.language }; // a feature-defined neutral shape — not the lib's full API
}
```

```ts
// features/products/formatters/toProductItem.ts — pure: locale IN, strings OUT (no global read)
import { isTopRated, type Product } from '../models/product';
export interface ProductItem { id: number; name: string; price: string; badge: string | null }

export const toProductItem = (p: Product, locale: string, t: (k: string) => string): ProductItem => ({
  id: p.id,
  name: p.name,
  price: new Intl.NumberFormat(locale, { style: 'currency', currency: 'USD' }).format(p.priceCents / 100),
  badge: isTopRated(p) ? t('product.badge.topRated') : null, // copy comes from t, never an inline literal
});
```

```ts
// features/products/viewmodels/useProductsViewModel.ts — threads locale/t from the VM into the formatter
export function useProductsViewModel(): ProductsVM {
  const { t, locale } = useTranslation();                      // the locale enters HERE; a switch re-renders the VM
  const { items, isLoading, isRefreshing, error, refetch } = useProductsData();
  const onRefresh = useCallback(() => refetch(), [refetch]);
  const viewItems = useMemo(() => items.map((p) => toProductItem(p, locale, t)), [items, locale, t]);

  if (isLoading) return { status: 'loading' };
  if (error && viewItems.length === 0) return { status: 'error', message: toErrorMessage(error, t), onRetry: onRefresh };
  if (viewItems.length === 0) return { status: 'empty', onRefresh };
  return { status: 'ready', items: viewItems, isRefreshing, onRefresh, banner: error ? toErrorMessage(error, t) : undefined };
}
```

The View ([section 7](triad-example.md#7-view--passive-only-branches-on-status-formats-nothing)) is **unchanged** — it still receives ready strings and decides nothing.
The formatter is unit-tested input→output by passing a fixed `locale`/`t` fake; the VM
test mocks `useTranslation` to assert the right keys reach the formatter. Swap the i18n
lib and **only `shared/i18n/index.ts` changes** — the same one-layer-swap claim as every
other boundary.

---

## 20. Accessibility — a View concern, but the *copy* and *announce-or-not* are the VM's

A11y splits cleanly along the triad (the Hygiene checklist in
[`mvvm-and-scaling.md`](mvvm-and-scaling.md) [section 2](mvvm-and-scaling.md#2-conformance-checklist-the-keep-it-faithful-core)): **how** an element is rendered/labelled
is the View's job; **what** the label *says* (user-facing copy) and **whether** a state
is announced are VM decisions, surfaced as ready values on the contract. The View adds
no logic — it spreads ready a11y props.

```ts
// features/products/viewmodels/ProductsVM.ts — the `ready` variant carries ready a11y copy
  | { status: 'ready'; items: ProductItem[]; isRefreshing: boolean; onRefresh: () => void;
      a11yRefreshHint: string;   // ready, localized — produced via the VM's t(), never inline in the View
      banner?: string };
```

```tsx
// features/products/views/ProductsView.tsx — the ready branch: roles/labels/touch targets, zero decisions
case 'ready':
  return (
    <FlatList
      data={vm.items}
      keyExtractor={(it) => String(it.id)}
      renderItem={({ item }) => <ProductRow item={item} />}
      refreshing={vm.isRefreshing}
      onRefresh={vm.onRefresh}
      // a11y is a View concern (how it's rendered); the *copy* came ready from the VM (what it says):
      refreshControl={
        <RefreshControl
          refreshing={vm.isRefreshing}
          onRefresh={vm.onRefresh}
          accessibilityLabel={vm.a11yRefreshHint}
        />
      }
    />
  );
```

```tsx
// features/products/components/ProductRow.tsx — semantic role, merged node, generous touch target
export const ProductRow = memo(function ProductRow({ item, onPress }: { item: ProductItem; onPress?: () => void }) {
  return (
    <Pressable
      onPress={onPress}
      accessible                                   // merge children into ONE screen-reader node
      accessibilityRole="button"
      accessibilityLabel={item.name}               // ready string from the formatter — never templated here
      accessibilityHint={item.badge ?? undefined}  // the formatter already localized it
      hitSlop={8}                                  // nudges the target toward ≥44pt iOS / 48dp Android
      style={{ minHeight: 48, justifyContent: 'center' }}
    >
      <Text>{item.name} — {item.price}{item.badge ? <Text> · {item.badge}</Text> : null}</Text>
    </Pressable>
  );
});
```

The View still **formats nothing and decides nothing** — every a11y *string* is a ready
value the VM produced (localized via [section 19](#19-localization-i18n--the-locale-enters-through-the-vm-formatters-stay-pure)); only roles, `hitSlop`, and layout (genuinely
presentational) live in the View. Reduce-motion (`AccessibilityInfo.isReduceMotionEnabled`)
gates the animation in [section 21](#21-animations--gestures--a-ui-hook-that-holds-no-data-never-the-viewmodel), which lives in a UI hook.

---

## 21. Animations / gestures — a UI hook that holds no data, never the ViewModel

Reanimated/Gesture Handler state is **UI state**, not screen state: it lives in a
**UI hook** (which holds no data — the Hooks rule in
[`mvvm-and-scaling.md`](mvvm-and-scaling.md) [section 1](mvvm-and-scaling.md#1-layer-responsibilities-the-contract)) or directly in the View, **never** in
the ViewModel (the VM imports no `react-native`). The hook also respects reduce-motion.

```ts
// shared/hooks/useFadeIn.ts — a UI hook: animation state only, no data, no VM coupling
import { useEffect } from 'react';
import { AccessibilityInfo } from 'react-native';
import { useSharedValue, useAnimatedStyle, withTiming } from 'react-native-reanimated';

export function useFadeIn(durationMs = 250) {
  const opacity = useSharedValue(0);
  useEffect(() => {
    AccessibilityInfo.isReduceMotionEnabled().then((reduced) => {
      opacity.value = reduced ? 1 : withTiming(1, { duration: durationMs }); // honor reduce-motion
    });
  }, [durationMs, opacity]);
  return useAnimatedStyle(() => ({ opacity: opacity.value })); // the View spreads this onto an Animated.View
}
```

```tsx
// the View owns the animation; the VM never sees it
const fadeStyle = useFadeIn();
return <Animated.View style={fadeStyle}><ProductRow item={item} /></Animated.View>;
```

The ViewModel is untouched: animation is presentation, so it stays on the View side of
the boundary. If an animation must be *triggered* by a domain event (e.g. "flash on
successful save"), the VM exposes a ready boolean/counter on the contract and the View's
UI hook reacts to it — the VM still imports no `react-native`.

---

## 22. Suspense — suspend at the query boundary, never from the ViewModel

The Suspense note in [`stack-choices.md`](stack-choices.md) in code: the neutral hook
**suspends** instead of returning `{ isLoading }`, an error boundary catches failures
instead of an `error` field, and the VM therefore **drops the `loading` variant** —
modelling only `empty`/`ready`. The View and tests don't change shape; the contract is
just shorter. **Suspend inside `queries/`, never in the VM.**

```ts
// features/products/queries/useProductsData.ts (Suspense) — suspends here, at the boundary
import { useSuspenseQuery } from '@tanstack/react-query'; // the ONLY place the lib is named
import { fetchProducts } from '../services/productService';
import type { Product } from '../models/product';

export interface ProductsData { items: Product[]; refetch: () => void } // no isLoading/error — Suspense owns those

export function useProductsData(): ProductsData {
  const q = useSuspenseQuery({ queryKey: ['products'], queryFn: fetchProducts }); // suspends until resolved
  const refetch = useCallback(() => { void q.refetch(); }, [q.refetch]);
  return { items: q.data, refetch };
}
```

```ts
// features/products/viewmodels/ProductsVM.ts (Suspense) — no `loading`, no `error` variant
export type ProductsVM =
  | { status: 'empty'; onRefresh: () => void }
  | { status: 'ready'; items: ProductItem[]; onRefresh: () => void };
```

```tsx
// the Screen provides the fallback + error boundary; the VM/View stay status-keyed
export function ProductsScreen() {
  return (
    <AppErrorBoundary>                               {/* catches the rejected query (replaces status:'error') */}
      <Suspense fallback={<Spinner accessibilityLabel="Loading products" />}>  {/* replaces status:'loading' */}
        <ProductsView {...useProductsViewModel()} />
      </Suspense>
    </AppErrorBoundary>
  );
}
```

Pick **one** mechanism per screen — `{ isLoading, error }` fields *or* Suspense — never
both for the same data. The View still only branches on `status`; it simply has fewer
branches to handle. Everything else (the formatter, the contract module, the fake-VM
test) is unchanged.

---

## 23. Referenced helpers & primitives (assumed, not re-implemented here)

The slices above import a handful of small primitives by name to stay focused on the
**boundaries** rather than on a design system. They are **not** files in this repo —
this is illustrative TypeScript, so you implement each one to taste (or pull it from
your own UI kit). Listed here once, with the signature each snippet assumes, so every
import resolves *conceptually* and nothing above is a dangling reference. Where a real
implementation already appears earlier, the section is noted.

**Presentational components** (`@/shared/components`) — dumb, props-in/JSX-out; they
format and decide nothing (that already happened in the VM):

| Component | Props it's given | Renders |
|---|---|---|
| `Spinner` | `accessibilityLabel: string` | a loading indicator (e.g. `ActivityIndicator`) |
| `ErrorState` | `message: string; onRetry: () => void` | an error message + a retry button |
| `EmptyState` | `message: string; onRefresh: () => void` | an empty-list placeholder + refresh |
| `Banner` | `message: string` | a soft inline notice above content still on screen |

```tsx
// shared/components/Spinner.tsx — the shape every "loading" branch assumes
import { ActivityIndicator } from 'react-native';
export const Spinner = ({ accessibilityLabel }: { accessibilityLabel: string }) =>
  <ActivityIndicator accessibilityLabel={accessibilityLabel} />;
// ErrorState / EmptyState / Banner follow the same pattern: ready props in, JSX out, zero decisions.
```

**Pure helpers** (formatters / parsers — pure functions, input→output):

| Helper | Signature | Lives in | Shown at |
|---|---|---|---|
| `formatPrice` | `(cents: number, locale?: string) => string` | `shared/formatters/` | used in [section 4](triad-example.md#4-formatter--domain--view-item-pure-display-ready); locale-threaded variant in [section 19](#19-localization-i18n--the-locale-enters-through-the-vm-formatters-stay-pure) |
| `toErrorMessage` | `(error: AppError, t?: TFn) => string` | `shared/formatters/` | maps the [section 1](triad-example.md#1-model--what-the-data-is--domain-rules) `AppError` union to ready copy |
| `parseEmail` | `(raw: string) => string` | `parsers/` | cleans input ([section 10](triad-advanced.md#10-controlled-inputs-forms--the-view-stays-dumb)) |
| `validateEmail` | `(raw: string) => 'invalid' \| 'empty' \| null` | `parsers/` | returns a typed **fault**, never a message ([section 10](triad-advanced.md#10-controlled-inputs-forms--the-view-stays-dumb)) |
| `toEmailErrorMessage` | `(fault: 'invalid' \| 'empty', t: TFn) => string` | `formatters/` | fault → localized copy ([section 10](triad-advanced.md#10-controlled-inputs-forms--the-view-stays-dumb)) |
| `assertNever` | `(x: never) => never` | `shared/` | **implemented** in [section 7](triad-example.md#7-view--passive-only-branches-on-status-formats-nothing) |

**Neutral hooks & services** (the boundary seams):

| Name | Signature | Boundary | Shown at |
|---|---|---|---|
| `useLoginMutation` | `() => { submit: (c: Credentials) => void; isSubmitting: boolean }` | `mutations/` (neutral shape, like [section 17](triad-advanced.md#17-mutations--the-write-path-behind-a-neutral-hook-invalidation--optimistic-update)) | [section 10](triad-advanced.md#10-controlled-inputs-forms--the-view-stays-dumb) |
| `useTranslation` | `() => { t: TFn; locale: string }` | `shared/i18n/` | **implemented** in [section 19](#19-localization-i18n--the-locale-enters-through-the-vm-formatters-stay-pure) |
| `fetchProduct` / `setFavorite` / `fetchProductsPage` | service functions returning domain types (the [section 2](triad-example.md#2-service--the-only-layer-that-touches-io-classifies-errors) pattern) | `services/` | single-item ([section 14](triad-advanced.md#14-typed-route-params--validated-at-the-boundary-the-vm-never-imports-the-router)), write ([section 17](triad-advanced.md#17-mutations--the-write-path-behind-a-neutral-hook-invalidation--optimistic-update)), paged ([section 18](triad-advanced.md#18-pagination--load-more--infinite-scroll-end-to-end)) |

`TFn` is the translate function `(key: string, vars?: Record<string, unknown>) => string`
from `useTranslation` ([section 19](#19-localization-i18n--the-locale-enters-through-the-vm-formatters-stay-pure)). The point of listing these rather than implementing them: the
**contract each presents to its caller** is what matters to the triad — the body is
yours, and swapping it never touches the VM or View.
