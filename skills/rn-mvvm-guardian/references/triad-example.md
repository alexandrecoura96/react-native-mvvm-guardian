# A worked triad in code

One minimal, end-to-end vertical slice — a **Products list** screen — showing
**every layer and the contract between them**. Illustrative TypeScript: adapt the
imports to your libraries. The point is the *shape* and the *boundaries*, not the
specific stack. Read [`mvvm-and-scaling.md`](mvvm-and-scaling.md) for the contract
this instantiates and [`conventions.md`](conventions.md) for the naming/structure.

> **Start at [section 0](#0-quickstart--the-smallest-faithful-slice-rung-1-40-lines), the quickstart (just below).**
> It is the smallest already-faithful slice (Model + View + ViewModel + Screen, one
> service, one formatter). **Every section after it only *splits* these same
> responsibilities into their own files when a distinct reason to change appears** —
> the full layered slice is the *destination*, not the day-one requirement. Don't read
> [sections 1–17](#1-model--what-the-data-is--domain-rules) as "everything you must build"; read them as "where each responsibility goes
> once it earns its own file" (the adoption ladder in [`conventions.md`](conventions.md)).

The dependency direction throughout: `Screen → ViewModel → (service / neutral hook
/ formatters)`; the **View depends only on the VM's contract**, never on the VM's
*implementation* (the hook) or any service. The contract lives in **its own module**
(`ProductsVM.ts`, [section 6](#6-viewmodel--the-views-contract-as-a-discriminated-union)) that both the View and the VM import, so the View never reaches
into the hook; and because it's a `import type`, it's erased at compile time — zero
runtime coupling either way.

---

## 0. Quickstart — the smallest faithful slice (Rung 1, ~40 lines)

Before the full layered slice below, this is the **minimum** that is already
MVVM-faithful: Model + Service + ViewModel + View + Screen, **co-located** in one
`screens/Products/` folder, **no** queries/store/parsers/transformer/formatter
files yet (the VM formats by calling one shared primitive). It's step 1 of the
adoption ladder ([`conventions.md`](conventions.md)) — copy it, then add layers
only when a distinct reason to change appears. Every later section just *splits*
these responsibilities into their own files; the boundaries are identical.

```ts
// screens/Products/useProductsViewModel.ts — state + "what to show"; no JSX, no react-native
import { useCallback, useEffect, useMemo, useState } from 'react';
import { fetchProducts } from '@/services/productService'; // the only layer that does I/O
import { formatPrice } from '@/formatters/formatPrice';     // a pure shared primitive

export type ProductsVM =
  | { status: 'loading' }
  | { status: 'error'; message: string; onRetry: () => void }
  | { status: 'ready'; items: { id: number; name: string; price: string }[] };

export function useProductsViewModel(): ProductsVM {
  const [raw, setRaw] = useState<{ id: number; name: string; priceCents: number }[] | null>(null);
  const [failed, setFailed] = useState(false);
  const [reloadKey, setReloadKey] = useState(0);
  useEffect(() => {
    const ctrl = new AbortController();
    setRaw(null); setFailed(false);
    fetchProducts({ signal: ctrl.signal })
      .then((items) => { if (!ctrl.signal.aborted) setRaw(items); })
      .catch(() => { if (!ctrl.signal.aborted) setFailed(true); });
    return () => ctrl.abort();
  }, [reloadKey]);
  const onRetry = useCallback(() => setReloadKey((k) => k + 1), []);
  const items = useMemo(
    () => (raw ?? []).map((p) => ({ id: p.id, name: p.name, price: formatPrice(p.priceCents) })),
    [raw],
  );
  if (failed) return { status: 'error', message: 'Could not load products', onRetry };
  if (raw === null) return { status: 'loading' };
  return { status: 'ready', items };
}
```
```tsx
// screens/Products/ProductsView.tsx — passive: branches on status, formats nothing
import { FlatList, Text } from 'react-native';
import { Spinner, ErrorState } from '@/shared/components';
import type { ProductsVM } from './useProductsViewModel'; // type-only — erased at compile time

export function ProductsView(vm: ProductsVM) {
  switch (vm.status) {
    case 'loading': return <Spinner accessibilityLabel="Loading products" />;
    case 'error':   return <ErrorState message={vm.message} onRetry={vm.onRetry} />;
    case 'ready':   return <FlatList data={vm.items} keyExtractor={(it) => String(it.id)}
                      renderItem={({ item }) => <Text>{item.name} — {item.price}</Text>} />;
    // once the union grows past these three, end on `default: assertNever(vm)` (see section 7)
    // so a forgotten variant fails to compile instead of rendering `undefined`.
  }
}
```
```tsx
// screens/Products/ProductsScreen.tsx — wiring only
import { useProductsViewModel } from './useProductsViewModel';
import { ProductsView } from './ProductsView';
export const ProductsScreen = () => <ProductsView {...useProductsViewModel()} />;
```

The inline strings here (`'Could not load products'`, `accessibilityLabel="Loading
products"`) are **illustrative, kept inline only for brevity** — under the
centralization + i18n rules a real app sources copy *and* accessibility text from the
VM or a `constants`/i18n module, never an inline literal in a View (see [section 7](#7-view--passive-only-branches-on-status-formats-nothing)).

That's the whole pattern. Note what is **already** true here, with no extra files:
the View formats and decides nothing, the VM holds no JSX, the Service is the only
I/O, and a fake VM can render the View in a test. The sections below show the same
slice once it has grown — a `transformer` once the wire shape diverges from the
domain, a `formatter`/view-item type once formatting is non-trivial, a `queries/`
hook once you need caching/refetch, and the contract split into its own module. At
Rung 1 the inline contract here (`type`-only imported by the View) is fine; split it
out as the feature grows (see [`conventions.md`](conventions.md)).

---

## 1. Model — what the data *is* + domain rules

```ts
// features/products/models/product.ts
export interface Product {
  id: number;
  name: string;
  priceCents: number; // domain keeps money as integer cents
  rating: number;     // 0..5
}

// A domain rule lives WITH the model — not in a mapper, not in the View:
export const isTopRated = (p: Product): boolean => p.rating >= 4.5;
```

```ts
// shared/errors.ts — a small domain-error taxonomy (see "Error handling" in mvvm-and-scaling.md)
export type AppError =
  | { kind: 'network' }            // offline / timeout
  | { kind: 'notFound' }
  | { kind: 'server'; status: number }
  | { kind: 'unknown' };
```

> **Thrown as a value — mind your lint preset.** Services `throw` an `AppError` ([section 2](#2-service--the-only-layer-that-touches-io-classifies-errors)),
> and it's a plain union, not an `Error` subclass. Under the `@typescript-eslint`
> *type-checked* preset (`only-throw-error`/`no-throw-literal`) that is flagged. Pick
> one and stay consistent: make `AppError` extend `Error` (`class AppError extends
> Error { constructor(readonly kind: …) { super(kind); } }`), or keep it a value union
> and either return a `Result<T, AppError>` from services instead of throwing, or scope
> the rule off for the classifier. The boundaries are identical either way — this is
> only about satisfying the **lint = 0** gate on your chosen preset.

## 2. Service — the only layer that touches I/O; classifies errors

```ts
// features/products/services/productService.ts
import { client } from '@/shared/api/client';            // the HTTP-client factory: the ONLY place the lib is imported
import { transformProduct, type ProductDTO } from '../transformers/transformProduct';
import { toAppError } from '@/shared/api/toAppError';
import type { Product } from '../models/product';

// Accepts an optional { signal } it forwards to client.get. TanStack passes a
// QueryFunctionContext (which carries its own `signal`) to the queryFn, so the
// query case (section 5) picks up cancellation for free; the no-lib case (section 12) passes
// an AbortController signal explicitly. Same signature serves both.
export async function fetchProducts(opts?: { signal?: AbortSignal }): Promise<Product[]> {
  try {
    const { data } = await client.get<{ products: ProductDTO[] }>('/products', { signal: opts?.signal });
    return data.products.map(transformProduct);          // wire → domain at the edge
  } catch (e) {
    throw toAppError(e);                                 // transport error → domain error
  }
}
```

### 2b. The HTTP client factory + the error classifier (the seam [section 2](#2-service--the-only-layer-that-touches-io-classifies-errors) depends on)

The two pieces [section 2](#2-service--the-only-layer-that-touches-io-classifies-errors) imports — `client` (the **only** place the HTTP lib is named) and
`toAppError` (transport/native failure → the domain `AppError` union of [section 1](#1-model--what-the-data-is--domain-rules)). Both
live in `shared/api/`, so swapping the HTTP lib touches *only* these files (see
`stack-choices.md` swappability). `toAppError` is **idempotent** — an `AppError` in
returns the same `AppError` out — so a service can classify an explicit HTTP-status
throw and re-throw through its single `catch` without double-wrapping (the
worked-examples [section 10b](worked-examples.md#10-the-same-recipes-on-a-contrasting-stack-bare-rn--react-navigation--redux-toolkit--fetch) fetch service relies on exactly this).

```ts
// shared/api/client.ts — the HTTP-client factory: the ONLY module that imports the lib
import axios, { type AxiosInstance } from 'axios';

export const client: AxiosInstance = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  timeout: 10_000,
});
// Auth headers/refresh are attached at boot via the AuthBridge port, not here —
// see worked-examples.md section 8d (so shared/api never imports a feature).
```

```ts
// shared/api/toAppError.ts — transport/native failure → the domain AppError union (section 1)
import axios from 'axios';
import type { AppError } from '@/shared/errors';

const APP_ERROR_KINDS = ['network', 'notFound', 'server', 'unknown'] as const;
const isAppError = (e: unknown): e is AppError =>
  typeof e === 'object' && e !== null && 'kind' in e &&
  (APP_ERROR_KINDS as readonly string[]).includes((e as { kind: unknown }).kind as string);

export function toAppError(e: unknown): AppError {
  if (isAppError(e)) return e;                       // idempotent: already classified
  if (axios.isAxiosError(e)) {
    if (e.response) {                                // server answered with a non-2xx
      return e.response.status === 404
        ? { kind: 'notFound' }
        : { kind: 'server', status: e.response.status };
    }
    return { kind: 'network' };                      // no response: offline / timeout
  }
  return { kind: 'unknown' };
}
```

On a `fetch` stack the classifier is the same shape but reads `e instanceof TypeError`
for the offline case and takes `{ status }` for the explicit HTTP-status throw — the
worked-examples.md [section 10b](worked-examples.md#10-the-same-recipes-on-a-contrasting-stack-bare-rn--react-navigation--redux-toolkit--fetch) version. Error classification always travels with the HTTP
client; layers above only ever see a domain `AppError`.

## 3. Transformer — wire → domain (pure)

```ts
// features/products/transformers/transformProduct.ts
import type { Product } from '../models/product';

export interface ProductDTO { id: number; title: string; price: number; rating?: number }

export const transformProduct = (dto: ProductDTO): Product => ({
  id: dto.id,
  name: dto.title,
  priceCents: Math.round(dto.price * 100),
  rating: dto.rating ?? 0, // fallback is a mapping concern
});
```

## 4. Formatter — domain → view-item (pure, display-ready)

```ts
// features/products/formatters/toProductItem.ts
import { isTopRated, type Product } from '../models/product';
import { formatPrice } from '@/shared/formatters/formatPrice';

export interface ProductItem {
  id: number;
  name: string;
  price: string;       // already display-ready, e.g. "$12.90"
  badge: string | null;
}

export const toProductItem = (p: Product): ProductItem => ({
  id: p.id,
  name: p.name,
  price: formatPrice(p.priceCents), // a shared primitive
  badge: isTopRated(p) ? 'Top rated' : null,
});
```

> **`formatPrice` is shown locale-free for brevity.** Money/number/date formatting is
> locale-dependent; in a localized app the VM threads the resolved `locale` in
> (`formatPrice(p.priceCents, locale)` → `Intl.NumberFormat(locale, …)`), exactly as
> the i18n note in [`mvvm-and-scaling.md`](mvvm-and-scaling.md) [section 2](mvvm-and-scaling.md#2-conformance-checklist-the-keep-it-faithful-core) prescribes for
> `toErrorMessage(error, t)`. The formatter stays pure (locale in → string out); only
> *where the locale comes from* (the VM) is fixed.

## 5. Neutral feature hook — wraps the server-state lib so the VM never sees it

The neutral hook **is the server-state boundary**, so it lives in `queries/`
(not `hooks/`, which holds UI/interaction hooks — see [`conventions.md`](conventions.md)).

```ts
// features/products/queries/useProductsData.ts
import { useCallback } from 'react';
import { useQuery } from '@tanstack/react-query'; // the ONLY place this lib is named
import { fetchProducts } from '../services/productService';
import type { Product } from '../models/product';
import type { AppError } from '@/shared/errors';

const EMPTY: Product[] = []; // module-level → a stable reference while there's no data

// This is the *canonical* neutral shape this material refers to elsewhere
// (stack-choices.md, section 15b). Defined inline here for one implementation; the moment a
// second implementation exists (the section 15b RTK Query swap), extract it to a sibling
// `queries/types.ts` and have both hooks import it — that's the `./types` section 15b imports.
export interface ProductsData {
  items: Product[];
  isLoading: boolean;
  isRefreshing: boolean;
  error: AppError | null;
  refetch: () => void;
}

// These field names are the *feature's* vocabulary, not the library's — they only
// happen to read like TanStack's because that's this slice's lib. Pick names by what
// the screen needs: if your lib has no `refetch`, expose `reload()`; if "refreshing"
// isn't a concept, drop it. The contract is yours to define — the lib adapts to it,
// not the reverse. That's exactly what makes the section 15b swap invisible to the VM.

// A feature-defined, lib-agnostic shape. Swap TanStack ↔ SWR ↔ RTK Query here only.
// (No server-state lib at all? See section 12 — the VM owns fetch + cancellation directly.)
export function useProductsData(): ProductsData {
  // Typing the generics (TError = AppError) makes q.error already AppError | null —
  // no unsafe cast. The service throws AppError, so this is honest, not a lie to TS.
  const q = useQuery<Product[], AppError>({ queryKey: ['products'], queryFn: fetchProducts });
  // q.refetch is referentially stable in TanStack Query; wrap once so the returned
  // refetch is stable too — an unstable handler here would defeat the VM's useCallback
  // and the memoized View downstream (the "stable references" Hygiene rule).
  const refetch = useCallback(() => { void q.refetch(); }, [q.refetch]);
  return {
    items: q.data ?? EMPTY,
    // q.isLoading (= isPending && isFetching), NOT q.isPending: a query gated with
    // `enabled: false` stays isPending forever (status 'pending', fetchStatus 'idle'),
    // so copying this hook for a gated detail/capability query (section 14) would show a
    // perpetual spinner. isLoading is false while disabled, true only on the first fetch.
    isLoading: q.isLoading,
    isRefreshing: q.isRefetching,
    error: q.error ?? null,
    refetch,
  };
}
```

## 6. ViewModel — the View's contract, as a **discriminated union**

The contract is a **standalone module** — the ViewModel's published interface, owned
by neither the hook nor the View. Both import it, so the View depends on the
*contract*, never on the VM's *implementation* (the hook). Each variant carries
**only** the data that state needs, so illegal combinations (e.g. `loading` *with*
items) are unrepresentable. This is the contract a fake VM satisfies in tests.

```ts
// features/products/viewmodels/ProductsVM.ts — the published contract (no logic; imports only the view-item type)
import type { ProductItem } from '../formatters/toProductItem';

export type ProductsVM =
  | { status: 'loading' }
  | { status: 'error'; message: string; onRetry: () => void }
  | { status: 'empty'; onRefresh: () => void }
  | { status: 'ready'; items: ProductItem[]; isRefreshing: boolean; onRefresh: () => void;
      banner?: string }; // soft error while data is still on screen (see the ordering below)
```

```ts
// features/products/viewmodels/useProductsViewModel.ts
import { useCallback, useMemo } from 'react';
import { useProductsData } from '../queries/useProductsData';
import { toProductItem } from '../formatters/toProductItem';
import { toErrorMessage } from '@/shared/formatters/toErrorMessage';
import type { ProductsVM } from './ProductsVM';

export function useProductsViewModel(): ProductsVM {
  const { items, isLoading, isRefreshing, error, refetch } = useProductsData();

  // stable references → a memoized View isn't re-rendered needlessly
  const onRefresh = useCallback(() => refetch(), [refetch]);
  // formatting is DELEGATED to the formatter, never done inline here.
  // Localized app: thread the locale from the VM — `items.map((p) => toProductItem(p, locale))`
  // and `toErrorMessage(error, t)` below; see the i18n note in mvvm-and-scaling.md section 2.
  // Omitted here only to keep the slice short.
  const viewItems = useMemo(() => items.map(toProductItem), [items]);

  if (isLoading) return { status: 'loading' };
  // error WITH cached data → keep the list, surface a soft banner; error with nothing to show → full error screen
  if (error && viewItems.length === 0) return { status: 'error', message: toErrorMessage(error), onRetry: onRefresh };
  if (viewItems.length === 0) return { status: 'empty', onRefresh };
  return { status: 'ready', items: viewItems, isRefreshing, onRefresh,
           banner: error ? toErrorMessage(error) : undefined };
}
```

## 7. View — passive: only branches on `status`, formats nothing

```tsx
// features/products/views/ProductsView.tsx
import { FlatList } from 'react-native';
import { Spinner, ErrorState, EmptyState, Banner } from '@/shared/components'; // shared presentational primitives
import { ProductRow } from '../components/ProductRow'; // a feature presentational component (defined below) — see the component tiers in worked-examples section 6
import { assertNever } from '@/shared/assertNever'; // exhaustiveness guard (defined below)
import type { ProductsVM } from '../viewmodels/ProductsVM'; // the contract MODULE, not the hook — nothing of the VM's implementation is imported

// Note: no service/store/query/navigation import; no .toFixed/date/template; no decisions.
export function ProductsView(vm: ProductsVM) {
  switch (vm.status) {
    case 'loading':
      return <Spinner accessibilityLabel="Loading products" />;
    case 'error':
      return <ErrorState message={vm.message} onRetry={vm.onRetry} />;
    case 'empty':
      return <EmptyState message="No products yet" onRefresh={vm.onRefresh} />;
    case 'ready':
      return (
        <>
          {vm.banner && <Banner message={vm.banner} />} {/* the VM decided; the View only renders */}
          <FlatList
            data={vm.items}
            keyExtractor={(it) => String(it.id)}
            renderItem={({ item }) => <ProductRow item={item} />} // ProductRow is memo()'d — see below
            refreshing={vm.isRefreshing}
            onRefresh={vm.onRefresh}
          />
        </>
      );
    default:
      return assertNever(vm); // a new variant the View forgot to handle fails to compile (TS)
                              // and throws instead of silently rendering `undefined` (JS / bad data)
  }
}
```

```ts
// shared/assertNever.ts — the exhaustiveness guard the Views above end on
export const assertNever = (x: never): never => {
  throw new Error(`Unhandled discriminated-union variant: ${JSON.stringify(x)}`);
};
```

The memoized row the `ready` branch renders — defined once, so an unchanged `item`
never re-renders (the "stable references + memoized rows" Hygiene rule):

```tsx
// features/products/components/ProductRow.tsx
import { memo } from 'react';
import { Text } from 'react-native';
import type { ProductItem } from '../formatters/toProductItem';

export const ProductRow = memo(function ProductRow({ item }: { item: ProductItem }) {
  // JSX children compose ready values — no string templating (`item.badge` is already
  // the display-ready string the formatter produced, or null). The separators are JSX
  // text, the same passive composition as section 0's `{item.name} — {item.price}`.
  return (
    <Text>
      {item.name} — {item.price}
      {item.badge ? <Text> · {item.badge}</Text> : null}
    </Text>
  );
});
```

> **Inline copy here is illustrative.** `"No products yet"` / `"Loading products"`
> keep the slice short; under the Hygiene rule (user-facing copy centralized, never
> in Views) a localized app sources these from the VM (a ready `message` on the
> variant, exactly as `error`/`banner` already do) or a `shared/constants`/i18n
> module — never an inline literal in the View. **This includes accessibility copy**
> (`accessibilityLabel`/`accessibilityHint`): it is user-facing text too, so a
> localized app sources it the same way — not as an inline literal.

> **The one hook the View may consume directly.** "No service/store/query/navigation
> import" still holds — but a **pure UI hook** (`use<Behavior>`, holds no data:
> animation, gesture, keyboard, scroll/focus, a visibility toggle) is presentation, so
> the View consumes it directly and the ViewModel never mediates it. [section 21](triad-crosscutting.md#21-animations--gestures--a-ui-hook-that-holds-no-data-never-the-viewmodel) shows the
> canonical case (an animation hook the View owns). The criterion is the hook's
> *nature* (presentation, not data) — keeping such state off the `<Screen>VM` contract
> is a consequence of that, not a reason to move screen state out of the VM.

## 8. Screen — the per-screen composition root

```tsx
// features/products/screens/ProductsScreen.tsx
import { useProductsViewModel } from '../viewmodels/useProductsViewModel';
import { ProductsView } from '../views/ProductsView';

export function ProductsScreen() {
  const vm = useProductsViewModel();   // wires the VM…
  return <ProductsView {...vm} />;      // …into the View. No state/JSX/logic of its own.
}
```

> **Spread vs. a single `vm` prop — pick one and be consistent.** `{...vm}` makes the
> VM contract *be* the View's props (clean, and what [section 0](#0-quickstart--the-smallest-faithful-slice-rung-1-40-lines)/[section 7](#7-view--passive-only-branches-on-status-formats-nothing) assume). On some TypeScript
> setups, spreading a **discriminated union** into JSX attributes can be rejected; if
> you hit that, pass the union as one prop instead — `<ProductsView vm={vm} />` with
> `function ProductsView({ vm }: { vm: ProductsVM })`. Both are faithful (the View
> still depends only on the contract); just don't mix the two styles within a project.
>
> **One performance nuance if you wrap the View in `React.memo`.** The VM returns a
> **new object every render**, so its *individual* fields are kept stable
> (`useCallback`/`useMemo`, the "stable references" Hygiene rule) but the top-level
> object identity is not. With `memo()`, **spread** preserves memoization (props are
> compared field-by-field, and those are stable), whereas a single `vm={vm}` prop
> defeats it (the object identity changes each render). If you memoize the View,
> prefer the spread — or memoize the `vm` object at the source.

## 9. Tests — the fake-VM View test + the renderHook VM test

```tsx
// View: render with a FAKE VM that satisfies the contract — no hook, no network.
it('renders the empty state', () => {
  const onRefresh = jest.fn();
  render(<ProductsView status="empty" onRefresh={onRefresh} />);
  expect(screen.getByText('No products yet')).toBeOnTheScreen();
});

// ViewModel: renderHook + a mocked data hook. The VM is a hook, not a pure function.
jest.mock('../queries/useProductsData');
it('exposes ready with formatted items', () => {
  (useProductsData as jest.Mock).mockReturnValue({
    items: [{ id: 1, name: 'Mug', priceCents: 1290, rating: 4.8 }],
    isLoading: false, isRefreshing: false, error: null, refetch: jest.fn(),
  });
  const { result } = renderHook(() => useProductsViewModel());
  expect(result.current.status).toBe('ready');
  if (result.current.status === 'ready') {
    expect(result.current.items[0].price).toBe('$12.90'); // formatter ran
    expect(result.current.items[0].badge).toBe('Top rated'); // domain rule ran
  }
});

// Pure layers are tested input→output, with no harness:
it('transforms wire → domain', () => {
  expect(transformProduct({ id: 1, title: 'Mug', price: 12.9 }))
    .toEqual({ id: 1, name: 'Mug', priceCents: 1290, rating: 0 });
});
```

---

> **Continue:** the **advanced cases** — controlled inputs/forms, Open/Closed via a registry,
> the no-server-state-lib path, the god-component refactor, typed route params, the same triad
> on MobX/RTK Query/Redux, an error boundary, mutations, and pagination — are in
> [`triad-advanced.md`](triad-advanced.md) (sections 10–18). The **cross-cutting concerns**
> (i18n, accessibility, animations/gestures, Suspense) and the **referenced-helpers appendix**
> are in [`triad-crosscutting.md`](triad-crosscutting.md) (sections 19–23). Same Products
> example, same boundaries throughout — only the file changes.
