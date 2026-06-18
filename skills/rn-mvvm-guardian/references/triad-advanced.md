# A worked triad in code — advanced cases (sections 10–18)

> Continues [`triad-example.md`](triad-example.md) (the core slice, sections 0–9) — same
> Products example, same boundaries. These sections work the harder cases: forms, Open/Closed
> via a registry, the no-server-state-lib path, the god-component refactor, typed route params,
> the same triad on MobX / RTK Query / Redux, an error boundary, mutations, and pagination.
> Cross-cutting concerns (i18n, accessibility, animations, Suspense) and the referenced-helpers
> appendix are in [`triad-crosscutting.md`](triad-crosscutting.md) (sections 19–23).

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
> controlled `setValue`/`watch` binding above lets the View stay the [section 10](#10-controlled-inputs-forms--the-view-stays-dumb) dumb component
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
HTTP client, so DIP holds. This is a drop-in replacement for [section 5](triad-example.md#5-neutral-feature-hook--wraps-the-server-state-lib-so-the-vm-never-sees-it) + [section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union): it returns the
**same `ProductsVM` contract**, so the View ([section 7](triad-example.md#7-view--passive-only-branches-on-status-formats-nothing)), the Screen ([section 8](triad-example.md#8-screen--the-per-screen-composition-root)), and the View test
([section 9](triad-example.md#9-tests--the-fake-vm-view-test--the-renderhook-vm-test)) are unchanged. Add TanStack/SWR later for caching/refetch/pagination and only
this file changes.

```ts
// features/products/viewmodels/useProductsViewModel.ts  (no server-state lib)
import { useCallback, useEffect, useMemo, useState } from 'react';
import { fetchProducts } from '../services/productService'; // accepts an optional { signal } it forwards to client.get
import { toProductItem } from '../formatters/toProductItem';
import { toErrorMessage } from '@/shared/formatters/toErrorMessage';
import type { Product } from '../models/product';
import type { AppError } from '@/shared/errors';
import type { ProductsVM } from './ProductsVM'; // the same published contract as section 6 — the View is unaffected by this swap

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

Testing this VM differs from [section 9](triad-example.md#9-tests--the-fake-vm-view-test--the-renderhook-vm-test) only in that the effect is async, so the assertion
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

> **Note — same contract, different *refresh* behavior.** The *types* are [section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union)'s, so
> the View, the Screen, and the View test don't change. But here `onRefresh` triggers
> a **full reload** (the screen returns to `loading`) and an error during reload
> **discards** the current list — this version does **not** realize [section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union)'s "soft banner
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
| `axios.get('/products')` | a **service** (the only place the HTTP lib is imported) | [section 2](triad-example.md#2-service--the-only-layer-that-touches-io-classifies-errors) |
| `r.data.products` → typed `Product[]` | a **transformer** (`transformProduct`) | [section 3](triad-example.md#3-transformer--wire--domain-pure) |
| `` `$${price.toFixed(2)}` `` | a **formatter** (`toProductItem` / `formatPrice`) | [section 4](triad-example.md#4-formatter--domain--view-item-pure-display-ready) |
| caching / loading / error lifecycle | a **neutral hook** (or the VM's effect, [section 12](#12-alternative--no-server-state-library-the-vm-owns-fetch--cancellation)) | [section 5](triad-example.md#5-neutral-feature-hook--wraps-the-server-state-lib-so-the-vm-never-sees-it) |
| deciding `loading`/`error`/`empty`/`ready` | the **ViewModel** (discriminated `status`) | [section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union) |
| `router.push(...)` | the **navigation facade**, called from the VM | — |
| rendering the resolved branch | the **View** (formats nothing, decides nothing) | [section 7](triad-example.md#7-view--passive-only-branches-on-status-formats-nothing) |
| wiring VM → View | the **Screen** | [section 8](triad-example.md#8-screen--the-per-screen-composition-root) |

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

The detail data hook is the single-item sibling of [section 5](triad-example.md#5-neutral-feature-hook--wraps-the-server-state-lib-so-the-vm-never-sees-it)'s `useProductsData` — same
neutral-shape rule, gated on the validated id so the lib's `enabled`/`useQuery` never
surface in the VM (`worked-examples.md` [section 3](worked-examples.md#3-rn-react-query--zustand-specifics-to-verify)):

```ts
// features/products/queries/useProductData.ts — neutral single-item hook (the section 5 shape, for one product)
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

// id may be null (an invalid route param, section 14) — gate the query on it INSIDE queries/,
// so the VM passes the value and never sees `skipToken`/`useQuery`. `skipToken` (TanStack
// v5) is the idiomatic gate: when id is null the query is disabled AND `id` narrows to
// `number` inside the queryFn — no `as` cast, no separate `enabled`. isLoading from
// q.isLoading (NOT q.isPending): a skipped query stays isPending forever (section 5).
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
// features/products/viewmodels/ProductDetailVM.ts — the published contract (same shape family as section 6)
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
  // id internally (`skipToken` — worked-examples section 3), so `skipToken`/`useQuery`
  // never surface in the VM. Keeping the gate inside `queries/` is what makes the section 15b
  // RTK Query swap invisible here.
  const { product, isLoading, error, refetch } = useProductData(id);
  // formatting DELEGATED; a localized app threads the locale in — `toProductItem(product, locale)`
  // and `toErrorMessage(error, t)` — exactly as section 6 (omitted here for brevity).
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

The triad is **stack-agnostic**: the View ([section 7](triad-example.md#7-view--passive-only-branches-on-status-formats-nothing)), its contract ([section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union)), and the Screen
([section 8](triad-example.md#8-screen--the-per-screen-composition-root)) don't change across stacks — only the VM's *internal* shape and which library
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
import type { ProductsVM } from './ProductsVM'; // the SAME contract module as section 6

export class ProductsViewModel {
  private products: Product[] | null = null;
  private error: AppError | null = null;
  private disposed = false; // race guard — the class analogue of section 12's AbortController

  // fake collaborators are injected in tests (DIP) — here, the section 2 service function
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
  // write to a torn-down VM — section 12 gets this for free from the effect's AbortController.
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
// the Screen consumes the class through observer — the View is the UNCHANGED section 7 component
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
  // explicit at the root, which is also what makes it testable with a fake — section 15a test.)
  const [vm] = useState(() => new ProductsViewModel({ fetchProducts }));
  useEffect(() => vm.dispose, [vm]); // cleanup-only wiring: cancel an in-flight load on unmount (section 15a guard)
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
neutral `ProductsData` shape ([section 5](triad-example.md#5-neutral-feature-hook--wraps-the-server-state-lib-so-the-vm-never-sees-it)), swapping the server-state lib only rewrites the
`queries/` hook; `useProductsViewModel` ([section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union)), the View, and the Screen are untouched.

```ts
// features/products/queries/useProductsData.ts (RTK Query) — same neutral ProductsData shape as section 5
import { useGetProductsQuery } from '@/shared/api/apiSlice'; // the RTK Query generated hook
import type { ProductsData } from './types';                 // the shared neutral shape, extracted from section 5

const EMPTY: ProductsData['items'] = []; // stable empty ref, typed via the neutral shape (matches section 5)

export function useProductsData(): ProductsData {
  // The API slice uses a custom baseQuery that classifies transport failures into
  // AppError (the section 2 job, moved into the baseQuery) — so `error` is already AppError | undefined,
  // no unsafe cast, just like section 5's honest typing.
  const { data, isLoading, isFetching, error, refetch } = useGetProductsQuery();
  // For a simple, non-paginated query `isFetching && !isLoading` coincides with section 5's
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
// features/products/viewmodels/useProductsViewModel.ts (Redux client state) — same ProductsVM as section 6
import { useCallback, useMemo } from 'react';
import { useSelector } from 'react-redux';
import { useAppDispatch } from '@/shared/stores/hooks'; // typed dispatch — accepts thunks (strict)
import { toProductItem } from '../formatters/toProductItem';
import { toErrorMessage } from '@/shared/formatters/toErrorMessage';
import { refreshProducts } from '../stores/productsThunks'; // a thunk: load + dispatch the actions above
import { selectProductsItems, selectProductsLoading, selectProductsError } from '../stores/productsSlice';
import type { ProductsVM } from './ProductsVM'; // the SAME contract module as section 6

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

That is the universality claim, made concrete: across the hook VM ([section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union)), the MobX
class ([section 15a](#15-the-same-triad-on-another-stack--a-mobx-class-vm-an-rtk-query-swap-and-a-redux-client-state-vm)), the RTK Query swap ([section 15b](#15-the-same-triad-on-another-stack--a-mobx-class-vm-an-rtk-query-swap-and-a-redux-client-state-vm)), and this Redux client-state VM, **the View
([section 7](triad-example.md#7-view--passive-only-branches-on-status-formats-nothing)), the contract ([section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union)), and the Screen ([section 8](triad-example.md#8-screen--the-per-screen-composition-root)) never change** — only the VM's internals
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

Queries ([section 5](triad-example.md#5-neutral-feature-hook--wraps-the-server-state-lib-so-the-vm-never-sees-it)) are the read path; **mutations are the write path**, and they live in the
same server-state boundary (`mutations/`). The rule is identical: the VM consumes a
**feature-defined neutral shape**, never the lib's `useMutation` result. Cache
invalidation and optimistic update/rollback are the *query layer's* job — they never
leak into the VM or View.

This slice assumes the [section 1](triad-example.md#1-model--what-the-data-is--domain-rules) `Product` model carries an `isFavorite: boolean` (a
favoritable product) — add it to the model + map it in the transformer like any other
domain field; it's elided from [section 1](triad-example.md#1-model--what-the-data-is--domain-rules) to keep that slice focused.

> **[section 12](#12-alternative--no-server-state-library-the-vm-owns-fetch--cancellation), [section 17](#17-mutations--the-write-path-behind-a-neutral-hook-invalidation--optimistic-update), and [section 18](#18-pagination--load-more--infinite-scroll-end-to-end) are independent illustrations — don't stack them blindly.** Each
> shows the *same* `ProductsVM` evolving along *one* axis: [section 12](#12-alternative--no-server-state-library-the-vm-owns-fetch--cancellation) drops to no server-state
> lib, [section 17](#17-mutations--the-write-path-behind-a-neutral-hook-invalidation--optimistic-update) adds favoriting, [section 18](#18-pagination--load-more--infinite-scroll-end-to-end) adds pagination. They each extend (or shrink) the
> `ready` variant differently, so the no-lib [section 12](#12-alternative--no-server-state-library-the-vm-owns-fetch--cancellation) VM and the [section 17](#17-mutations--the-write-path-behind-a-neutral-hook-invalidation--optimistic-update) mutation fields are
> *not* meant to coexist as-is — composing both would leave [section 12](#12-alternative--no-server-state-library-the-vm-owns-fetch--cancellation) returning a `ready`
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
  // object `m` (recreated each render), so `toggle` stays stable (the section 5 "stable references" rule).
  const toggle = useCallback((id: number, next: boolean) => mutate({ id, next }), [mutate]);
  return { toggle, isToggling: isPending, error: error ?? null };
}
```

The VM consumes it exactly like a query hook — a ready handler and a ready boolean,
no lib types. First extend the [section 6](triad-example.md#6-viewmodel--the-views-contract-as-a-discriminated-union) contract's `ready` variant (the other variants are
unchanged); then wire the hook in the VM and return the two fields on the ready branch:

```ts
// features/products/viewmodels/ProductsVM.ts — the `ready` variant of section 6, extended (others unchanged)
  | { status: 'ready'; items: ProductItem[]; isRefreshing: boolean; onRefresh: () => void;
      onToggleFavorite: (id: number, next: boolean) => void; // ready handler — the View just calls it
      isTogglingFavorite: boolean;                           // ready boolean — the View just reads it
      banner?: string };
```

```ts
// features/products/viewmodels/useProductsViewModel.ts — the section 6 VM, with the mutation wired in
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

The View ([section 7](triad-example.md#7-view--passive-only-branches-on-status-formats-nothing)) reads `vm.onToggleFavorite`/`vm.isTogglingFavorite` on the ready branch
and decides nothing — the same passive contract, one variant richer.

> **Swapping the lib stays a one-layer change.** On SWR it's `useSWRMutation` +
> `mutate(key)`; on RTK Query the generated `useToggleFavoriteMutation` + a manual
> optimistic `updateQueryData` in `onQueryStarted`. Whatever the lib, `useToggleFavorite`
> returns the **same `ToggleFavorite` shape**, so the VM and View never change — the
> same universality claim as [section 15](#15-the-same-triad-on-another-stack--a-mobx-class-vm-an-rtk-query-swap-and-a-redux-client-state-vm). The mutation is tested by mocking the service and
> asserting the optimistic patch + rollback on a rejected call.

---

## 18. Pagination — load-more / infinite scroll, end to end

Pagination is just two more fields on the **neutral hook** and two more on the VM's
`ready` variant. The lib's `useInfiniteQuery` (or its `loadMore`/`hasNextPage`) stays
caged in `queries/`; the VM sees a flat list + intent-named controls; the View branches
on them and **formats nothing**. The discriminant stays `ready` — "loading the next
page" is a *secondary* state inside `ready`, not a top-level variant (data is on screen).

```ts
// features/products/queries/useProductsData.ts — the section 5 hook, paginated (lib caged here)
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
// features/products/viewmodels/useProductsViewModel.ts — the section 6 VM, pagination wired in
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
// features/products/views/ProductsView.tsx — the ready branch only (others as section 7)
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
> `useGetProductsQuery` with merged pages, or the [section 12](#12-alternative--no-server-state-library-the-vm-owns-fetch--cancellation) no-lib path (the VM holds a
> `cursor` in state and the service takes it) all return the **same `ProductsData`
> shape** — VM and View never change. Tested by faking the hook: assert the footer
> spinner toggles with `isLoadingMore` and `onEndReached` is gated by `hasMore`.