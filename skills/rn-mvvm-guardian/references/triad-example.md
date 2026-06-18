# A worked triad in code

One minimal, end-to-end vertical slice — a **Products list** screen — showing
**every layer and the contract between them**. Illustrative TypeScript: adapt the
imports to your libraries. The point is the *shape* and the *boundaries*, not the
specific stack. Read [`mvvm-and-scaling.md`](mvvm-and-scaling.md) for the contract
this instantiates and [`conventions.md`](conventions.md) for the naming/structure.

> **Start at §0, the quickstart (just below).**
> It is the smallest already-faithful slice (Model + View + ViewModel + Screen, one
> service, one formatter). **Every section after it only *splits* these same
> responsibilities into their own files when a distinct reason to change appears** —
> the full layered slice is the *destination*, not the day-one requirement. Don't read
> §1–§17 as "everything you must build"; read them as "where each responsibility goes
> once it earns its own file" (the adoption ladder in [`conventions.md`](conventions.md)).

The dependency direction throughout: `Screen → ViewModel → (service / neutral hook
/ formatters)`; the **View depends only on the VM's contract**, never on the VM's
*implementation* (the hook) or any service. The contract lives in **its own module**
(`ProductsVM.ts`, §6) that both the View and the VM import, so the View never reaches
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
    // once the union grows past these three, end on `default: assertNever(vm)` (see §7)
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
VM or a `constants`/i18n module, never an inline literal in a View (see §7).

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

> **Thrown as a value — mind your lint preset.** Services `throw` an `AppError` (§2),
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
// query case (§5) picks up cancellation for free; the no-lib case (§12) passes
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

### 2b. The HTTP client factory + the error classifier (the seam §2 depends on)

The two pieces §2 imports — `client` (the **only** place the HTTP lib is named) and
`toAppError` (transport/native failure → the domain `AppError` union of §1). Both
live in `shared/api/`, so swapping the HTTP lib touches *only* these files (see
`stack-choices.md` swappability). `toAppError` is **idempotent** — an `AppError` in
returns the same `AppError` out — so a service can classify an explicit HTTP-status
throw and re-throw through its single `catch` without double-wrapping (the
worked-examples §10b fetch service relies on exactly this).

```ts
// shared/api/client.ts — the HTTP-client factory: the ONLY module that imports the lib
import axios, { type AxiosInstance } from 'axios';

export const client: AxiosInstance = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  timeout: 10_000,
});
// Auth headers/refresh are attached at boot via the AuthBridge port, not here —
// see worked-examples.md §8d (so shared/api never imports a feature).
```

```ts
// shared/api/toAppError.ts — transport/native failure → the domain AppError union (§1)
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
worked-examples.md §10b version. Error classification always travels with the HTTP
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
> the i18n note in [`mvvm-and-scaling.md`](mvvm-and-scaling.md) §2 prescribes for
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
// (stack-choices.md, §15b). Defined inline here for one implementation; the moment a
// second implementation exists (the §15b RTK Query swap), extract it to a sibling
// `queries/types.ts` and have both hooks import it — that's the `./types` §15b imports.
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
// not the reverse. That's exactly what makes the §15b swap invisible to the VM.

// A feature-defined, lib-agnostic shape. Swap TanStack ↔ SWR ↔ RTK Query here only.
// (No server-state lib at all? See §12 — the VM owns fetch + cancellation directly.)
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
    // so copying this hook for a gated detail/capability query (§14) would show a
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
  // and `toErrorMessage(error, t)` below; see the i18n note in mvvm-and-scaling.md §2.
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
import { ProductRow } from '../components/ProductRow'; // a feature presentational component (defined below) — see the component tiers in worked-examples §6
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
  // text, the same passive composition as §0's `{item.name} — {item.price}`.
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
> VM contract *be* the View's props (clean, and what §0/§7 assume). On some TypeScript
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

## 10. Controlled inputs (forms) — the View stays dumb

The VM owns field state, parsing, and validation; the View only binds `value` +
`onChangeText` and renders ready errors. (The form library, if any, lives inside
the VM — see [`stack-choices.md`](stack-choices.md).) This simple form uses flat
ready fields; when a submission blocks the *whole* screen, model it as the
top-level `status: 'submitting'` variant from [`conventions.md`](conventions.md)
instead. Note the VM exposes a **ready** `isSubmitDisabled` boolean — the View
decides nothing, it just reads it.

```ts
export type LoginVM = {
  email: string;
  emailError: string | null;   // a ready, localized message — produced by a *formatter* (see the VM), never by the parser
  password: string;
  isSubmitDisabled: boolean;    // VM-computed (invalid OR already submitting) — the View just reads it
  isSubmitting: boolean;        // for the in-button spinner only
  onChangeEmail: (t: string) => void;
  onChangePassword: (t: string) => void;
  onSubmit: () => void;
};
```

```tsx
// The View parses/validates nothing itself:
<TextInput value={vm.email} onChangeText={vm.onChangeEmail} accessibilityLabel="Email" />
{vm.emailError && <HelperText type="error">{vm.emailError}</HelperText>}
<Button title="Sign in" onPress={vm.onSubmit} disabled={vm.isSubmitDisabled} />
```

The VM owns the whole flow end-to-end — field state, validation (a `parser`
returning a typed *fault*, then a `formatter` for the localized message), the ready
`isSubmitDisabled` boolean, and the submit (a `mutation`). The View reads the result
and decides nothing:

```ts
// features/auth/viewmodels/useLoginViewModel.ts
import { useCallback, useMemo, useState } from 'react';
import { useLoginMutation } from '../mutations/useLoginMutation';        // server-state layer — neutral shape { submit, isSubmitting }
import { parseEmail, validateEmail } from '../parsers/email';            // parser: cleans + returns a typed FAULT, never a display string
import { toEmailErrorMessage } from '../formatters/toEmailErrorMessage'; // fault → localized copy (a formatter, not the parser)
import { useTranslation } from '@/shared/i18n';                         // neutral i18n hook — the locale enters through the VM
import type { LoginVM } from './LoginVM';

export function useLoginViewModel(): LoginVM {
  const { t } = useTranslation();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const { submit, isSubmitting } = useLoginMutation();                  // neutral hook — the VM never sees the lib's mutate/isPending
  const emailFault = useMemo(() => validateEmail(email), [email]);      // 'invalid' | 'empty' | null — a domain value, never validated inline
  const emailError = useMemo(                                           // localization is the formatter's job, with locale threaded from the VM
    () => (emailFault ? toEmailErrorMessage(emailFault, t) : null),
    [emailFault, t],
  );
  const isSubmitDisabled = emailFault !== null || password.length === 0 || isSubmitting;
  const onSubmit = useCallback(() => {
    if (isSubmitDisabled) return;
    submit({ email: parseEmail(email), password });                     // parser cleans, mutation submits
  }, [isSubmitDisabled, email, password, submit]);
  return {
    email, emailError, password, isSubmitDisabled, isSubmitting,
    onChangeEmail: setEmail, onChangePassword: setPassword, onSubmit,
  };
}
```

When a submission blocks the *whole* screen instead of just the button, drop the flat
fields for the top-level `status: 'submitting'` variant from
[`conventions.md`](conventions.md) — same rule, different granularity.

### With a form library (react-hook-form) — the lib lives *inside* the VM

A form library is an **implementation detail of the ViewModel**, never a View concern.
The VM owns `useForm`; the exported contract is the **same `LoginVM`**, so the View
(and its test) are byte-for-byte unchanged. Domain-rule validation still goes through
the `parser`/model — not an ad-hoc resolver in the View — and the message still comes
from a `formatter`:

```ts
// features/auth/viewmodels/useLoginViewModel.ts — react-hook-form, same LoginVM contract
import { useForm } from 'react-hook-form';
import { useLoginMutation } from '../mutations/useLoginMutation';
import { validateEmail } from '../parsers/email';            // the rule lives in the parser/model, not RHF
import { toEmailErrorMessage } from '../formatters/toEmailErrorMessage';
import { useTranslation } from '@/shared/i18n';
import type { LoginVM } from './LoginVM';

export function useLoginViewModel(): LoginVM {
  const { t } = useTranslation();
  const { watch, setValue, handleSubmit } = useForm({ defaultValues: { email: '', password: '' } });
  const { submit, isSubmitting } = useLoginMutation();
  const email = watch('email');
  const password = watch('password');
  const emailFault = validateEmail(email);                   // 'invalid' | 'empty' | null
  const emailError = emailFault ? toEmailErrorMessage(emailFault, t) : null; // formatter owns the copy

  return {
    email, emailError, password,
    isSubmitDisabled: emailFault !== null || password.length === 0 || isSubmitting,
    isSubmitting,
    onChangeEmail: (v) => setValue('email', v),              // controlled binding keeps the View passive…
    onChangePassword: (v) => setValue('password', v),
    onSubmit: handleSubmit(({ email, password }) => submit({ email, password })),
  };
}
```

> **Keep `Controller` out of the View** (the `stack-choices.md` form nuance). The
> controlled `setValue`/`watch` binding above lets the View stay the §10 dumb component
> with no form-lib import. If you prefer RHF's uncontrolled `Controller` for perf, wrap
> it in a **feature presentational component** fed ready `value`/`error`/`onChange` from
> the VM — never let a `Controller` in the View carry a domain rule or formatting.

---

## 11. Open/Closed in practice — extend a registry, don't edit a `switch`

A new screen/feature should not force an edit to a central file. Prefer a registry
the new unit *extends*:

```ts
// ❌ Closed-for-extension: every new screen edits this switch (OCP violation)
function iconFor(screen: string) {
  switch (screen) { case 'products': return '🛍'; case 'cart': return '🛒'; /* …edit forever */ }
}

// ✅ Open: each feature contributes its own entry; the registry isn't modified
// features/products/index.ts
export const productsTab = { name: 'products', icon: '🛍', screen: ProductsScreen };
// app/tabs.ts  — composes contributions; adding a feature doesn't edit a shared switch
export const tabs = [productsTab, cartTab /* , … */];
```

---

## 12. Alternative — no server-state library (the VM owns fetch + cancellation)

The documented "small app" case (see [`stack-choices.md`](stack-choices.md)): no
`queries/` hook — the VM calls the `service` directly and owns loading / error /
cancellation in local state. The `service` is **still** the abstraction over the
HTTP client, so DIP holds. This is a drop-in replacement for §5 + §6: it returns the
**same `ProductsVM` contract**, so the View (§7), the Screen (§8), and the View test
(§9) are unchanged. Add TanStack/SWR later for caching/refetch/pagination and only
this file changes.

```ts
// features/products/viewmodels/useProductsViewModel.ts  (no server-state lib)
import { useCallback, useEffect, useMemo, useState } from 'react';
import { fetchProducts } from '../services/productService'; // accepts an optional { signal } it forwards to client.get
import { toProductItem } from '../formatters/toProductItem';
import { toErrorMessage } from '@/shared/formatters/toErrorMessage';
import type { Product } from '../models/product';
import type { AppError } from '@/shared/errors';
import type { ProductsVM } from './ProductsVM'; // the same published contract as §6 — the View is unaffected by this swap

export function useProductsViewModel(): ProductsVM {
  const [products, setProducts] = useState<Product[] | null>(null);
  const [error, setError] = useState<AppError | null>(null);
  const [reloadKey, setReloadKey] = useState(0);

  useEffect(() => {
    const ctrl = new AbortController();          // guard the race: cancel on unmount/reload
    setProducts(null);
    setError(null);
    fetchProducts({ signal: ctrl.signal })
      .then((items) => { if (!ctrl.signal.aborted) setProducts(items); })
      .catch((e: AppError) => { if (!ctrl.signal.aborted) setError(e); });
    return () => ctrl.abort();                   // bumping reloadKey re-runs this, aborting the prior request
  }, [reloadKey]);

  const onRefresh = useCallback(() => setReloadKey((k) => k + 1), []);
  const viewItems = useMemo(() => (products ?? []).map(toProductItem), [products]);

  if (error) return { status: 'error', message: toErrorMessage(error), onRetry: onRefresh };
  if (products === null) return { status: 'loading' };
  if (viewItems.length === 0) return { status: 'empty', onRefresh };
  return { status: 'ready', items: viewItems, isRefreshing: false, onRefresh };
}
```

The race guard is the point: every effect run owns an `AbortController`, the cleanup
aborts it, and `if (!signal.aborted)` keeps a stale response from overwriting fresh
state. This is exactly what a server-state lib gives you for free — which is *why*
you graduate to one once caching/refetch/pagination become real needs.

Testing this VM differs from §9 only in that the effect is async, so the assertion
waits for the resolved state — still via the contract, no network:

```tsx
// the service is the seam (it owns I/O), so mock IT — the VM is exercised whole
jest.mock('../services/productService');
it('resolves to ready with formatted items', async () => {
  (fetchProducts as jest.Mock).mockResolvedValue(
    [{ id: 1, name: 'Mug', priceCents: 1290, rating: 4.8 }],
  );
  const { result } = renderHook(() => useProductsViewModel());
  expect(result.current.status).toBe('loading');                  // first render, before the effect resolves
  await waitFor(() => expect(result.current.status).toBe('ready')); // effect settled
  if (result.current.status === 'ready') {
    expect(result.current.items[0].price).toBe('$12.90');         // formatter ran
  }
});

it('resolves to error when the service rejects', async () => {
  (fetchProducts as jest.Mock).mockRejectedValue({ kind: 'network' } satisfies AppError);
  const { result } = renderHook(() => useProductsViewModel());
  await waitFor(() => expect(result.current.status).toBe('error'));
});
```

> **Note — same contract, different *refresh* behavior.** The *types* are §6's, so
> the View, the Screen, and the View test don't change. But here `onRefresh` triggers
> a **full reload** (the screen returns to `loading`) and an error during reload
> **discards** the current list — this version does **not** realize §6's "soft banner
> with the data kept on screen". To keep data visible during a refresh *without* a
> lib, don't null `products` until the new response arrives (preserve the last value
> and expose `isRefreshing: true` meanwhile). Getting that behavior for free is
> precisely why you graduate to a server-state lib — the contract is identical, only
> the UX quality differs.

---

## 13. Anti-pattern — the "god component" that does everything, refactored

The shape to recognize in a review: **one component that fetches, formats, decides
state, and navigates** — every layer's job collapsed into the View. It violates SRP
(many reasons to change), DIP (imports the HTTP client + router directly), and the
passive-View rule (formats + decides) all at once.

```tsx
// ❌ features/products/ProductsScreen.tsx — a god component (do NOT do this)
import { FlatList, Text } from 'react-native';
import axios from 'axios';                 // HTTP client in a View (transport leak)
import { router } from 'expo-router';      // router in a View (navigation leak)

export function ProductsScreen() {
  const [products, setProducts] = useState<any[] | null>(null);   // `any`, no domain type
  const [error, setError] = useState(false);
  useEffect(() => { axios.get('/products')                        // fetch in the View
    .then((r) => setProducts(r.data.products)).catch(() => setError(true)); }, []);

  if (error) return <Text>Something went wrong</Text>;            // View decides error state
  if (!products) return <Text>Loading…</Text>;                    // View decides loading state
  return (
    <FlatList
      data={products}
      keyExtractor={(p) => p.id}
      renderItem={({ item }) => (
        <Text onPress={() => router.push(`/product/${item.id}`)}> {/* navigation in the View */}
          {item.title} — ${(item.price).toFixed(2)}               {/* formatting in the View */}
        </Text>
      )}
    />
  );
}
```

**The fix is not "tidy this file" — it's to put each reason-to-change in its layer**,
which is exactly the slice already shown above:

| What the god component did | Where it belongs | Section |
|---|---|---|
| `axios.get('/products')` | a **service** (the only place the HTTP lib is imported) | §2 |
| `r.data.products` → typed `Product[]` | a **transformer** (`transformProduct`) | §3 |
| `` `$${price.toFixed(2)}` `` | a **formatter** (`toProductItem` / `formatPrice`) | §4 |
| caching / loading / error lifecycle | a **neutral hook** (or the VM's effect, §12) | §5 |
| deciding `loading`/`error`/`empty`/`ready` | the **ViewModel** (discriminated `status`) | §6 |
| `router.push(...)` | the **navigation facade**, called from the VM | — |
| rendering the resolved branch | the **View** (formats nothing, decides nothing) | §7 |
| wiring VM → View | the **Screen** | §8 |

Same screen, same behavior — now eight pieces each with one reason to change, each
testable in isolation. That is the whole point of the triad.

---

## 14. Typed route params — validated at the boundary, the VM never imports the router

A detail screen reads `id` from the route. The rule from
[`stack-choices.md`](stack-choices.md) holds: **route params come through the
navigation layer and are validated by a `parser`** — the ViewModel never imports the
router. The param hook lives in the feature's `navigation.ts` (the one place the
router's param hook is named), so the VM depends on a neutral, already-validated value.

```ts
// features/products/parsers/parseRouteId.ts — clean + validate inbound params (pure)
export const parseRouteId = (raw: string | string[] | undefined): number | null => {
  const s = Array.isArray(raw) ? raw[0] : raw; // a route param may arrive as string[]
  const n = Number(s);
  return Number.isInteger(n) && n > 0 ? n : null; // invalid → null, never a thrown error
};
```

```ts
// features/products/navigation.ts — feature-owned routing; the only place the router is named outside app/ route files
import { router, useLocalSearchParams } from 'expo-router';
import { parseRouteId } from './parsers/parseRouteId';

// typed route (object form), not a raw template string — keeps expo-router's route typing
export const openProduct = (id: number) => router.push({ pathname: '/product/[id]', params: { id } });

// A neutral, validated param hook. The VM depends on THIS, not on expo-router.
export function useProductRouteId(): number | null {
  const { id } = useLocalSearchParams<{ id: string }>();
  return parseRouteId(id);
}
```

The detail data hook is the single-item sibling of §5's `useProductsData` — same
neutral-shape rule, gated on the validated id so the lib's `enabled`/`useQuery` never
surface in the VM (`worked-examples.md` §3):

```ts
// features/products/queries/useProductData.ts — neutral single-item hook (the §5 shape, for one product)
import { useCallback } from 'react';
import { skipToken, useQuery } from '@tanstack/react-query'; // the ONLY place this lib is named
import { fetchProduct } from '../services/productService';
import type { Product } from '../models/product';
import type { AppError } from '@/shared/errors';

export interface ProductData {
  product: Product | null;
  isLoading: boolean;
  error: AppError | null;
  refetch: () => void;
}

// id may be null (an invalid route param, §14) — gate the query on it INSIDE queries/,
// so the VM passes the value and never sees `skipToken`/`useQuery`. `skipToken` (TanStack
// v5) is the idiomatic gate: when id is null the query is disabled AND `id` narrows to
// `number` inside the queryFn — no `as` cast, no separate `enabled`. isLoading from
// q.isLoading (NOT q.isPending): a skipped query stays isPending forever (§5).
export function useProductData(id: number | null): ProductData {
  const q = useQuery<Product, AppError>({
    queryKey: ['product', id],
    queryFn: id === null ? skipToken : () => fetchProduct(id),
  });
  const refetch = useCallback(() => { void q.refetch(); }, [q.refetch]);
  return { product: q.data ?? null, isLoading: q.isLoading, error: q.error ?? null, refetch };
}
```

```ts
// features/products/viewmodels/ProductDetailVM.ts — the published contract (same shape family as §6)
import type { ProductItem } from '../formatters/toProductItem';

export type ProductDetailVM =
  | { status: 'loading' }
  | { status: 'error'; message: string; onRetry: () => void }
  | { status: 'ready'; item: ProductItem }; // a detail screen shows one product, not a list
```

```ts
// features/products/viewmodels/useProductDetailViewModel.ts
import { useMemo } from 'react';
import { useProductRouteId } from '../navigation'; // neutral value — no router import here
import { useProductData } from '../queries/useProductData';
import { toProductItem } from '../formatters/toProductItem';
import { toErrorMessage } from '@/shared/formatters/toErrorMessage';
import { appNavigation } from '@/shared/navigation/appNavigation'; // the facade — VMs call it, never the router
import type { ProductDetailVM } from './ProductDetailVM';

export function useProductDetailViewModel(): ProductDetailVM {
  const id = useProductRouteId();                 // already validated (number | null)
  // The VM passes the *value*, not a lib flag: the neutral hook gates itself on a null
  // id internally (`skipToken` — worked-examples §3), so `skipToken`/`useQuery`
  // never surface in the VM. Keeping the gate inside `queries/` is what makes the §15b
  // RTK Query swap invisible here.
  const { product, isLoading, error, refetch } = useProductData(id);
  // formatting DELEGATED; a localized app threads the locale in — `toProductItem(product, locale)`
  // and `toErrorMessage(error, t)` — exactly as §6 (omitted here for brevity).
  const item = useMemo(() => (product ? toProductItem(product) : null), [product]);

  // an invalid route id has nothing to refetch, so the error variant's action goes back
  // through the facade — a *meaningful* handler, never a no-op (every contract field must do
  // something). Message via a formatter, never an inline literal.
  if (id === null) return { status: 'error', message: toErrorMessage({ kind: 'notFound' }), onRetry: appNavigation.goBack };
  if (isLoading) return { status: 'loading' };
  if (error || item === null) return { status: 'error', message: toErrorMessage(error ?? { kind: 'notFound' }), onRetry: refetch };
  return { status: 'ready', item };
}
```

The parser is unit-tested input→output like every pure layer; the VM is tested via
its contract with the param hook mocked. Swap expo-router ↔ React Navigation and only
`navigation.ts` changes — the parser and the VM don't.

---

## 15. The same triad on another stack — a MobX class VM, an RTK Query swap, and a Redux client-state VM

The triad is **stack-agnostic**: the View (§7), its contract (§6), and the Screen
(§8) don't change across stacks — only the VM's *internal* shape and which library
sits in a layer do. Three contrasts:

**a) MobX — the ViewModel is an observable class** (not a hook). It still produces the
**same `ProductsVM` discriminated union** the View consumes, and is tested by
instantiating it with fake collaborators.

```ts
// features/products/viewmodels/ProductsViewModel.ts (MobX) — an observable CLASS
import { makeAutoObservable, runInAction } from 'mobx';
import { toProductItem } from '../formatters/toProductItem';
import { toErrorMessage } from '@/shared/formatters/toErrorMessage';
import type { Product } from '../models/product';
import type { AppError } from '@/shared/errors';
import type { ProductsVM } from './ProductsVM'; // the SAME contract module as §6

export class ProductsViewModel {
  private products: Product[] | null = null;
  private error: AppError | null = null;
  private disposed = false; // race guard — the class analogue of §12's AbortController

  // fake collaborators are injected in tests (DIP) — here, the §2 service function
  constructor(private readonly deps: { fetchProducts: () => Promise<Product[]> }) {
    makeAutoObservable(this);
    void this.load();
  }
  private async load() {
    runInAction(() => { this.error = null; });
    try {
      const products = await this.deps.fetchProducts();
      if (this.disposed) return;                        // a stale response must not overwrite state after teardown
      runInAction(() => { this.products = products; }); // mutate observables inside an action — required after every await
    } catch (e) {
      if (this.disposed) return;
      runInAction(() => { this.error = e as AppError; });
    }
  }
  onRefresh = () => { void this.load(); };
  // the Screen calls this on unmount (a useEffect cleanup) so an in-flight load can't
  // write to a torn-down VM — §12 gets this for free from the effect's AbortController.
  dispose = () => { this.disposed = true; };

  // a computed returning the contract — formatting still DELEGATED to the formatter
  get state(): ProductsVM {
    if (this.error) return { status: 'error', message: toErrorMessage(this.error), onRetry: this.onRefresh };
    if (this.products === null) return { status: 'loading' };
    const items = this.products.map(toProductItem);
    if (items.length === 0) return { status: 'empty', onRefresh: this.onRefresh };
    return { status: 'ready', items, isRefreshing: false, onRefresh: this.onRefresh };
  }
}
```

```tsx
// the Screen consumes the class through observer — the View is the UNCHANGED §7 component
import { observer } from 'mobx-react-lite';
import { useEffect, useState } from 'react';
import { fetchProducts } from '../services/productService';
import { ProductsView } from '../views/ProductsView';
import { ProductsViewModel } from '../viewmodels/ProductsViewModel';

export const ProductsScreen = observer(() => {
  // useState lazy-init, NOT useMemo: the VM instance is state that MUST persist —
  // React may discard a useMemo, recreating the VM and losing its state mid-session.
  // The Screen importing `fetchProducts` to inject it is composition-root wiring, not
  // logic — the "Screen is wiring only" rule bans state/JSX/decisions, not DI. (The
  // hook VM hides this by importing its deps itself; a class VM makes the injection
  // explicit at the root, which is also what makes it testable with a fake — §15a test.)
  const [vm] = useState(() => new ProductsViewModel({ fetchProducts }));
  useEffect(() => vm.dispose, [vm]); // cleanup-only wiring: cancel an in-flight load on unmount (§15a guard)
  return <ProductsView {...vm.state} />; // reading vm.state here makes the Screen reactive
});
```

```ts
// MobX VM test — instantiate with a fake, assert the exposed contract (no renderHook needed)
it('exposes ready with formatted items', async () => {
  const vm = new ProductsViewModel({
    fetchProducts: jest.fn().mockResolvedValue([{ id: 1, name: 'Mug', priceCents: 1290, rating: 4.8 }]),
  });
  await waitFor(() => expect(vm.state.status).toBe('ready'));
  if (vm.state.status === 'ready') expect(vm.state.items[0].price).toBe('$12.90'); // formatter ran
});
```

**b) RTK Query / Redux — the VM doesn't change at all.** Because the VM depends on the
neutral `ProductsData` shape (§5), swapping the server-state lib only rewrites the
`queries/` hook; `useProductsViewModel` (§6), the View, and the Screen are untouched.

```ts
// features/products/queries/useProductsData.ts (RTK Query) — same neutral ProductsData shape as §5
import { useGetProductsQuery } from '@/shared/api/apiSlice'; // the RTK Query generated hook
import type { ProductsData } from './types';                 // the shared neutral shape, extracted from §5

const EMPTY: ProductsData['items'] = []; // stable empty ref, typed via the neutral shape (matches §5)

export function useProductsData(): ProductsData {
  // The API slice uses a custom baseQuery that classifies transport failures into
  // AppError (the §2 job, moved into the baseQuery) — so `error` is already AppError | undefined,
  // no unsafe cast, just like §5's honest typing.
  const { data, isLoading, isFetching, error, refetch } = useGetProductsQuery();
  // For a simple, non-paginated query `isFetching && !isLoading` coincides with §5's
  // `isRefetching`. With pagination it would also be true during load-more — derive
  // `isRefreshing`/`isLoadingMore` separately then (from the infinite-query/slice state).
  return { items: data ?? EMPTY, isLoading, isRefreshing: isFetching && !isLoading, error: error ?? null, refetch };
}
```

**c) Redux Toolkit *client* state — the VM consumes a selector + dispatches intents.**
The contrast with (b): there the server-state lib changed; here the *client*-state lib
holds the data. The selector is the boundary (ISP — the VM subscribes to a minimal
slice, never the whole store), `dispatch(intent())` is the only write, and the VM
still produces the **same `ProductsVM`** the View consumes. Cross-concern teardown
would go to a coordinator (or a thunk/saga — see [`stack-choices.md`](stack-choices.md)),
never a reducer.

```ts
// features/products/stores/productsSlice.ts — pure state container; selectors are the boundary
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';
import type { Product } from '../models/product';
import type { AppError } from '@/shared/errors';

interface ProductsState { items: Product[]; isLoading: boolean; error: AppError | null }
const initialState: ProductsState = { items: [], isLoading: false, error: null };

const slice = createSlice({
  name: 'products',
  initialState,
  reducers: {
    loadStarted: (s) => { s.isLoading = true; s.error = null; },
    loadSucceeded: (s, a: PayloadAction<Product[]>) => { s.isLoading = false; s.items = a.payload; },
    loadFailed: (s, a: PayloadAction<AppError>) => { s.isLoading = false; s.error = a.payload; },
  },
});
export const { loadStarted, loadSucceeded, loadFailed } = slice.actions;
export default slice.reducer;

// selectors = the minimal slices the VM subscribes to (ISP), never the whole store
export const selectProductsItems = (s: { products: ProductsState }) => s.products.items;
export const selectProductsLoading = (s: { products: ProductsState }) => s.products.isLoading;
export const selectProductsError = (s: { products: ProductsState }) => s.products.error;
```

```ts
// shared/stores/store.ts — the typed seam every Redux consumer depends on (strict: no untyped dispatch)
import { configureStore, type ThunkAction, type Action } from '@reduxjs/toolkit';
import products from '@/features/products/stores/productsSlice';

export const store = configureStore({ reducer: { products } });
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;                  // knows about the thunk middleware
export type AppThunk = ThunkAction<void, RootState, unknown, Action>;

// shared/stores/hooks.ts — the typed hooks the VMs use (RTK's recommended pattern)
import { useDispatch } from 'react-redux';
export const useAppDispatch = () => useDispatch<AppDispatch>();   // accepts thunks; plain useDispatch() would not
```

```ts
// features/products/stores/productsThunks.ts — orchestrates load + dispatches the actions above
import { fetchProducts } from '../services/productService'; // the service is still the only I/O
import { loadStarted, loadSucceeded, loadFailed } from './productsSlice';
import type { AppThunk } from '@/shared/stores/store'; // the app's typed thunk alias (the store.ts block above)
import type { AppError } from '@/shared/errors';

// A plain thunk (redux-thunk ships with Redux Toolkit), typed via the app's AppThunk
// alias so it stays strict (no loose `(a: unknown) => void` dispatch). It orchestrates;
// the reducers stay pure and the service still owns I/O + error classification.
export const refreshProducts = (): AppThunk => async (dispatch) => {
  dispatch(loadStarted());
  try {
    dispatch(loadSucceeded(await fetchProducts()));
  } catch (e) {
    dispatch(loadFailed(e as AppError));
  }
};
```

```ts
// features/products/viewmodels/useProductsViewModel.ts (Redux client state) — same ProductsVM as §6
import { useCallback, useMemo } from 'react';
import { useSelector } from 'react-redux';
import { useAppDispatch } from '@/shared/stores/hooks'; // typed dispatch — accepts thunks (strict)
import { toProductItem } from '../formatters/toProductItem';
import { toErrorMessage } from '@/shared/formatters/toErrorMessage';
import { refreshProducts } from '../stores/productsThunks'; // a thunk: load + dispatch the actions above
import { selectProductsItems, selectProductsLoading, selectProductsError } from '../stores/productsSlice';
import type { ProductsVM } from './ProductsVM'; // the SAME contract module as §6

export function useProductsViewModel(): ProductsVM {
  const items = useSelector(selectProductsItems);     // minimal slice — not useSelector((s) => s)
  const isLoading = useSelector(selectProductsLoading);
  const error = useSelector(selectProductsError);
  const dispatch = useAppDispatch();
  const onRefresh = useCallback(() => { void dispatch(refreshProducts()); }, [dispatch]);
  const viewItems = useMemo(() => items.map(toProductItem), [items]); // formatting still DELEGATED

  if (isLoading) return { status: 'loading' };
  if (error && viewItems.length === 0) return { status: 'error', message: toErrorMessage(error), onRetry: onRefresh };
  if (viewItems.length === 0) return { status: 'empty', onRefresh };
  return { status: 'ready', items: viewItems, isRefreshing: false, onRefresh,
           banner: error ? toErrorMessage(error) : undefined };
}
```

That is the universality claim, made concrete: across the hook VM (§6), the MobX
class (§15a), the RTK Query swap (§15b), and this Redux client-state VM, **the View
(§7), the contract (§6), and the Screen (§8) never change** — only the VM's internals
and which library sits in a layer do. **A library swap is a one-layer change.** If a
swap forced the VM or View to change, a boundary would be leaking.

---

## 16. Error boundary — render-time crashes, contained at the navigator level

The discriminated-`status` flow above handles **expected** failures (a fetch that
threw a classified `AppError`, surfaced as `status: 'error'` and testable through the
contract). **Unexpected render-time exceptions** are a separate axis: contain them
with an **error boundary registered at the navigator/`app/` level** (or a dedicated
`shared/components` wrapper) — never as inline JSX inside a Screen, which would break
the "Screen is wiring only" rule.

```tsx
// shared/components/AppErrorBoundary.tsx — a class component (the only React API that catches render errors)
import { Component, type ReactNode } from 'react';
import { ErrorState } from '@/shared/components';
import { reportError } from '@/shared/services/reporter'; // crash reporting fires here / a thin service — never a View

export class AppErrorBoundary extends Component<{ children: ReactNode }, { hasError: boolean }> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error: Error) { reportError(error); }
  render() {
    return this.state.hasError
      ? <ErrorState message="Something went wrong" onRetry={() => this.setState({ hasError: false })} />
      : this.props.children;
  }
}
// app/_layout.tsx wraps the route tree once: <AppErrorBoundary><Slot /></AppErrorBoundary>
```

> Two axes, two mechanisms: the VM's `status: 'error'` for **expected** domain errors
> (recoverable, contract-tested), the boundary for **unexpected** render crashes. The
> message string here is illustrative — a localized app sources it like any other copy
> (see the centralization rule and the i18n note in
> [`mvvm-and-scaling.md`](mvvm-and-scaling.md)).

---

## 17. Mutations — the write path, behind a neutral hook (invalidation + optimistic update)

Queries (§5) are the read path; **mutations are the write path**, and they live in the
same server-state boundary (`mutations/`). The rule is identical: the VM consumes a
**feature-defined neutral shape**, never the lib's `useMutation` result. Cache
invalidation and optimistic update/rollback are the *query layer's* job — they never
leak into the VM or View.

This slice assumes the §1 `Product` model carries an `isFavorite: boolean` (a
favoritable product) — add it to the model + map it in the transformer like any other
domain field; it's elided from §1 to keep that slice focused.

> **§12, §17, and §18 are independent illustrations — don't stack them blindly.** Each
> shows the *same* `ProductsVM` evolving along *one* axis: §12 drops to no server-state
> lib, §17 adds favoriting, §18 adds pagination. They each extend (or shrink) the
> `ready` variant differently, so the no-lib §12 VM and the §17 mutation fields are
> *not* meant to coexist as-is — composing both would leave §12 returning a `ready`
> that's missing `onToggleFavorite`. The lesson is the *pattern* ("add only the ready
> fields your screen actually exposes, behind a neutral hook"), not a single cumulative
> contract. Build the variant your screen needs; ignore the fields it doesn't.

```ts
// features/products/mutations/useToggleFavorite.ts — the ONLY place the lib's useMutation is named
import { useCallback } from 'react';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { setFavorite } from '../services/productService'; // the service does the I/O + error classification
import type { Product } from '../models/product';
import type { AppError } from '@/shared/errors';

// A feature-defined, lib-agnostic shape — the VM sees this, not TanStack's mutation result.
export interface ToggleFavorite {
  toggle: (id: number, next: boolean) => void;
  isToggling: boolean;
  error: AppError | null;
}

// The 4th generic (TContext) MUST be spelled out: with only the first three given, it
// defaults to `unknown` (it is NOT inferred from onMutate), so `ctx?.prev` below wouldn't
// type-check. Naming it { prev: Product[] | undefined } keeps the rollback honest — no cast.
type FavoriteCtx = { prev: Product[] | undefined };

export function useToggleFavorite(): ToggleFavorite {
  const qc = useQueryClient();
  const { mutate, isPending, error } = useMutation<void, AppError, { id: number; next: boolean }, FavoriteCtx>({
    mutationFn: ({ id, next }) => setFavorite(id, next),
    // optimistic update: snapshot → patch the cache → roll back on error (all in the query layer)
    onMutate: async ({ id, next }) => {
      await qc.cancelQueries({ queryKey: ['products'] });
      const prev = qc.getQueryData<Product[]>(['products']);
      qc.setQueryData<Product[]>(['products'], (old) =>
        (old ?? []).map((p) => (p.id === id ? { ...p, isFavorite: next } : p)));
      return { prev }; // becomes the typed `ctx` handed to onError
    },
    onError: (_err, _vars, ctx) => { if (ctx?.prev) qc.setQueryData(['products'], ctx.prev); }, // rollback
    onSettled: () => { void qc.invalidateQueries({ queryKey: ['products'] }); }, // reconcile with the server
  });
  // `mutate` is referentially stable in TanStack Query — depend on IT, not the whole result
  // object `m` (recreated each render), so `toggle` stays stable (the §5 "stable references" rule).
  const toggle = useCallback((id: number, next: boolean) => mutate({ id, next }), [mutate]);
  return { toggle, isToggling: isPending, error: error ?? null };
}
```

The VM consumes it exactly like a query hook — a ready handler and a ready boolean,
no lib types. First extend the §6 contract's `ready` variant (the other variants are
unchanged); then wire the hook in the VM and return the two fields on the ready branch:

```ts
// features/products/viewmodels/ProductsVM.ts — the `ready` variant of §6, extended (others unchanged)
  | { status: 'ready'; items: ProductItem[]; isRefreshing: boolean; onRefresh: () => void;
      onToggleFavorite: (id: number, next: boolean) => void; // ready handler — the View just calls it
      isTogglingFavorite: boolean;                           // ready boolean — the View just reads it
      banner?: string };
```

```ts
// features/products/viewmodels/useProductsViewModel.ts — the §6 VM, with the mutation wired in
export function useProductsViewModel(): ProductsVM {
  const { items, isLoading, isRefreshing, error, refetch } = useProductsData();
  const { toggle, isToggling } = useToggleFavorite();        // the neutral mutation hook — no lib types
  const onRefresh = useCallback(() => refetch(), [refetch]);
  const onToggleFavorite = useCallback(                      // stable ref → memoized View isn't re-rendered
    (id: number, next: boolean) => toggle(id, next),
    [toggle],
  );
  const viewItems = useMemo(() => items.map(toProductItem), [items]);

  if (isLoading) return { status: 'loading' };
  if (error && viewItems.length === 0) return { status: 'error', message: toErrorMessage(error), onRetry: onRefresh };
  if (viewItems.length === 0) return { status: 'empty', onRefresh };
  return { status: 'ready', items: viewItems, isRefreshing, onRefresh,
           onToggleFavorite, isTogglingFavorite: isToggling,
           banner: error ? toErrorMessage(error) : undefined };
}
```

The View (§7) reads `vm.onToggleFavorite`/`vm.isTogglingFavorite` on the ready branch
and decides nothing — the same passive contract, one variant richer.

> **Swapping the lib stays a one-layer change.** On SWR it's `useSWRMutation` +
> `mutate(key)`; on RTK Query the generated `useToggleFavoriteMutation` + a manual
> optimistic `updateQueryData` in `onQueryStarted`. Whatever the lib, `useToggleFavorite`
> returns the **same `ToggleFavorite` shape**, so the VM and View never change — the
> same universality claim as §15. The mutation is tested by mocking the service and
> asserting the optimistic patch + rollback on a rejected call.

---

## 18. Pagination — load-more / infinite scroll, end to end

Pagination is just two more fields on the **neutral hook** and two more on the VM's
`ready` variant. The lib's `useInfiniteQuery` (or its `loadMore`/`hasNextPage`) stays
caged in `queries/`; the VM sees a flat list + intent-named controls; the View branches
on them and **formats nothing**. The discriminant stays `ready` — "loading the next
page" is a *secondary* state inside `ready`, not a top-level variant (data is on screen).

```ts
// features/products/queries/useProductsData.ts — the §5 hook, paginated (lib caged here)
import { useInfiniteQuery } from '@tanstack/react-query'; // still the ONLY place the lib is named
// ... fetchProductsPage(pageParam) returns { items: Product[]; nextCursor: number | null }

export interface ProductsData {
  items: Product[];
  isLoading: boolean;
  isRefreshing: boolean;
  isLoadingMore: boolean;   // a page is in flight (data already on screen)
  hasMore: boolean;         // there is a next page to request
  error: AppError | null;
  refetch: () => void;
  loadMore: () => void;     // feature vocabulary — not the lib's fetchNextPage
}

export function useProductsData(): ProductsData {
  const q = useInfiniteQuery<{ items: Product[]; nextCursor: number | null }, AppError>({
    queryKey: ['products'],
    queryFn: ({ pageParam }) => fetchProductsPage(pageParam as number),
    initialPageParam: 0,
    getNextPageParam: (last) => last.nextCursor, // null → no more pages
  });
  const items = useMemo(() => q.data?.pages.flatMap((p) => p.items) ?? EMPTY, [q.data]);
  const loadMore = useCallback(() => { if (q.hasNextPage) void q.fetchNextPage(); }, [q.hasNextPage, q.fetchNextPage]);
  const refetch = useCallback(() => { void q.refetch(); }, [q.refetch]);
  return {
    items,
    isLoading: q.isLoading,
    isRefreshing: q.isRefetching,
    isLoadingMore: q.isFetchingNextPage,
    hasMore: q.hasNextPage,
    error: q.error ?? null,
    refetch, loadMore,
  };
}
```

```ts
// features/products/viewmodels/ProductsVM.ts — the `ready` variant, extended (others unchanged)
  | { status: 'ready'; items: ProductItem[]; isRefreshing: boolean; onRefresh: () => void;
      isLoadingMore: boolean;   // ready boolean — the View shows a footer spinner
      hasMore: boolean;         // ready boolean — the View gates the "load more" affordance
      onEndReached: () => void; // ready handler — the View just calls it (onEndReached / button press)
      banner?: string };
```

```ts
// features/products/viewmodels/useProductsViewModel.ts — the §6 VM, pagination wired in
export function useProductsViewModel(): ProductsVM {
  const { items, isLoading, isRefreshing, isLoadingMore, hasMore, error, refetch, loadMore } = useProductsData();
  const onRefresh = useCallback(() => refetch(), [refetch]);
  const onEndReached = useCallback(() => loadMore(), [loadMore]); // stable ref → memoized list isn't re-rendered
  const viewItems = useMemo(() => items.map(toProductItem), [items]);

  if (isLoading) return { status: 'loading' };
  if (error && viewItems.length === 0) return { status: 'error', message: toErrorMessage(error), onRetry: onRefresh };
  if (viewItems.length === 0) return { status: 'empty', onRefresh };
  return { status: 'ready', items: viewItems, isRefreshing, onRefresh,
           isLoadingMore, hasMore, onEndReached,
           banner: error ? toErrorMessage(error) : undefined };
}
```

The View wires the list to `onEndReached`, shows a footer spinner while `isLoadingMore`,
and renders nothing more once `!hasMore` — it reads booleans the VM resolved and decides
nothing:

```tsx
// features/products/views/ProductsView.tsx — the ready branch only (others as §7)
<FlatList
  data={vm.items}
  keyExtractor={(it) => String(it.id)}
  renderItem={({ item }) => <ProductRow item={item} />}
  onEndReachedThreshold={0.5}
  onEndReached={vm.hasMore ? vm.onEndReached : undefined} // gate at the edge, not inside the handler
  ListFooterComponent={vm.isLoadingMore ? <ActivityIndicator /> : null}
/>
```

> **Swapping the lib stays a one-layer change.** SWR's `useSWRInfinite`, an RTK Query
> `useGetProductsQuery` with merged pages, or the §12 no-lib path (the VM holds a
> `cursor` in state and the service takes it) all return the **same `ProductsData`
> shape** — VM and View never change. Tested by faking the hook: assert the footer
> spinner toggles with `isLoadingMore` and `onEndReached` is gated by `hasMore`.

---

## 19. Localization (i18n) — the locale enters through the VM, formatters stay pure

The prose rule (the i18n note in [`mvvm-and-scaling.md`](mvvm-and-scaling.md) §2) in
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

The View (§7) is **unchanged** — it still receives ready strings and decides nothing.
The formatter is unit-tested input→output by passing a fixed `locale`/`t` fake; the VM
test mocks `useTranslation` to assert the right keys reach the formatter. Swap the i18n
lib and **only `shared/i18n/index.ts` changes** — the same one-layer-swap claim as every
other boundary.

---

## 20. Accessibility — a View concern, but the *copy* and *announce-or-not* are the VM's

A11y splits cleanly along the triad (the Hygiene checklist in
[`mvvm-and-scaling.md`](mvvm-and-scaling.md) §2): **how** an element is rendered/labelled
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
value the VM produced (localized via §19); only roles, `hitSlop`, and layout (genuinely
presentational) live in the View. Reduce-motion (`AccessibilityInfo.isReduceMotionEnabled`)
gates the animation in §21, which lives in a UI hook.

---

## 21. Animations / gestures — a UI hook that holds no data, never the ViewModel

Reanimated/Gesture Handler state is **UI state**, not screen state: it lives in a
**UI hook** (which holds no data — the Hooks rule in
[`mvvm-and-scaling.md`](mvvm-and-scaling.md) §1) or directly in the View, **never** in
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
