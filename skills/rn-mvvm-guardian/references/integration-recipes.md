# Integration recipes — the previously-deferred concerns, worked

The concerns [`conventions.md`](conventions.md) and [`stack-choices.md`](stack-choices.md)
flag as *"compatible with the contract but not prescribed in depth"* — now worked at the
same discipline as the rest of the material: **the library lives in exactly one layer
behind a neutral contract, and the ViewModel / View never change when you swap it.**

Every recipe here answers the one question this skill exists to protect — *does this keep
the new capability on the right side of a boundary?* — by reusing the foundations rather
than inventing new layers:

- the triad in code ([`triad-example.md`](triad-example.md) §0–§9),
- the neutral data hook (`use<Thing>Data`, triad §5),
- the error taxonomy (`AppError`, triad §1),
- typed route params validated by a parser (triad §14),
- the `AuthBridge` `api ⇄ auth` inversion ([`worked-examples.md`](worked-examples.md) §8d),
- the navigation facade + coordinator ([`worked-examples.md`](worked-examples.md) §8a/§8c).

**Read those first.** The libraries named below (Sentry, Apollo, NetInfo, …) are *instances*,
exactly like every entry in [`stack-choices.md`](stack-choices.md) — picked to make the code
concrete, never recommended or required. Where a library's own API churns across majors, the
recipe says so; the **boundary**, not the lib, is the durable part.

---

## 1. Observability — analytics, logging, crash reporting (fired from the VM, never the View)

**The rule.** Telemetry is an *intent the ViewModel expresses* (a domain event happened); the
View renders and forwards events — it never reports. A thin `services/` reporter behind a
neutral `Reporter` contract is the **only** place the SDK is named, and **PII never reaches a
log line**. This is the same `reporter` the error boundary already calls
([`triad-example.md`](triad-example.md) §16) — completed here.

```ts
// shared/services/reporter.ts — the ONLY module that names the telemetry SDK
import * as Sentry from '@sentry/react-native'; // an instance — swap for PostHog/Firebase/a home-grown service behind this contract
import type { AppError } from '@/shared/errors';

// the neutral contract the VM depends on (DIP) — never the SDK's surface
export interface Reporter {
  track: (event: string, props?: Record<string, string | number | boolean>) => void;
  report: (error: AppError | Error, context?: Record<string, unknown>) => void;
}

// illustrative: a real impl redacts known PII keys (emails/tokens/names) before anything leaves the device
const scrubPII = (ctx?: Record<string, unknown>): Record<string, unknown> | undefined => ctx;

export const reporter: Reporter = {
  track: (event, props) => Sentry.captureMessage(event, { level: 'info', extra: props }),
  report: (error, context) => Sentry.captureException(error, { extra: scrubPII(context) }),
};

// the alias the render-time error boundary (triad §16) calls:
export const reportError = (e: Error) => reporter.report(e);
```

```ts
// the VM expresses intent — the View just calls the ready handler and reports nothing
export function useCheckoutViewModel(): CheckoutVM {
  // …
  const onPay = useCallback(() => {
    reporter.track('checkout_started', { items: cart.length }); // intent, in the VM — never PII
    submit();
  }, [cart.length, submit]);
  // …
}
```

The View imports no reporter; it has no idea telemetry exists. Swapping Sentry → PostHog →
a custom service rewrites **only** `reporter.ts`. For test-time substitution, inject a fake
`Reporter` the same way the VM injects any collaborator (`mvvm-and-scaling.md` §1 DI seam), or
`jest.mock('@/shared/services/reporter')`.

> **Why it protects the contract.** The View stays passive (no side effects), the VM owns the
> *decision to report*, and the SDK stays caged — one reason to change, one layer.

---

## 2. Real-time — WebSocket / SSE / subscriptions (a `services/` stream adapter feeding the cache)

**The rule.** A live connection is *server-state lifecycle*, so it lives in the server-state
boundary, **not** the View. A `services/` stream adapter (core `WebSocket` — no library needed)
pushes domain values into the query cache (or a store); the VM consumes the **same neutral shape**
it would for a fetched list, so it never learns the data is live.

```ts
// features/chat/services/messageStream.ts — the stream adapter; core WebSocket, no lib
import { transformMessage, type MessageDTO } from '../transformers/transformMessage';
import type { Message } from '../models/message';

// returns an unsubscribe fn; wire→domain happens at the edge, exactly like a fetch service (triad §2)
export function openMessageStream(roomId: string, onMessage: (m: Message) => void): () => void {
  const ws = new WebSocket(`${process.env.EXPO_PUBLIC_WS_URL}/rooms/${roomId}`);
  ws.onmessage = (e) => onMessage(transformMessage(JSON.parse(e.data) as MessageDTO));
  return () => ws.close();
}
```

```ts
// features/chat/queries/useMessagesData.ts — the stream feeds the cache; the VM sees the §5 neutral shape
import { useEffect } from 'react';
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { fetchMessages } from '../services/messageService';
import { openMessageStream } from '../services/messageStream';
import type { Message } from '../models/message';
import type { AppError } from '@/shared/errors';

const EMPTY: Message[] = [];

export function useMessagesData(roomId: string): { items: Message[]; isLoading: boolean; error: AppError | null } {
  const q = useQuery<Message[], AppError>({ queryKey: ['messages', roomId], queryFn: () => fetchMessages(roomId) });
  const qc = useQueryClient();
  // the subscription lifecycle lives HERE (server-state layer), the analogue of useQuery owning the fetch —
  // not in the VM, and never in the View. Live messages merge into the same cache the initial fetch filled.
  useEffect(() => openMessageStream(roomId, (m) =>
    qc.setQueryData<Message[]>(['messages', roomId], (old) => [...(old ?? []), m])),
    [roomId, qc]);
  return { items: q.data ?? EMPTY, isLoading: q.isLoading, error: q.error ?? null };
}
```

The VM consumes `useMessagesData(roomId)` like any list hook — it never imports `WebSocket`.
**SSE**: swap `WebSocket` for `EventSource`. **GraphQL subscriptions**: the client's `subscribe`
(see §4). **Connection scoped to a screen** instead of the cache? Lift the effect into a neutral
`useMessageStream` hook the VM owns — still never the View. The boundary is identical.

---

## 3. Offline-first / sync — a persistence adapter behind the cache + a sync coordinator

**The rule.** Two pieces, both behind boundaries: **(a)** persist the query cache to durable
storage so reads work offline; **(b)** a **sync coordinator** flushes queued writes when
connectivity returns. The VM reads the **same domain types** and never knows whether data came
from disk or network. Conflict resolution is *domain logic* → a model use-case, never a View/VM.

```ts
// shared/persistence/queryPersister.ts — back the query cache with durable storage (the persistence boundary)
import { persistQueryClient } from '@tanstack/react-query-persist-client';
import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister';
import type { QueryClient } from '@tanstack/react-query';
import { storage } from './kvStore'; // the ONLY place the storage lib is named (MMKV/AsyncStorage behind this)

export function enableOfflineCache(queryClient: QueryClient): void {
  persistQueryClient({ queryClient, persister: createAsyncStoragePersister({ storage }) });
}
```

```ts
// shared/coordinators/syncCoordinator.ts — flush queued writes on reconnect; holds no UI state
import NetInfo from '@react-native-community/netinfo'; // connectivity stays behind this coordinator
import type { QueryClient } from '@tanstack/react-query';

export function startSync(queryClient: QueryClient): () => void {
  // returns an unsubscribe fn; resumePausedMutations replays writes queued while offline
  return NetInfo.addEventListener((s) => { if (s.isConnected) void queryClient.resumePausedMutations(); });
}
```

`enableOfflineCache` + `startSync` are wired once at boot (the composition root), like
`setupAuthInterceptors`. The VM is untouched — offline-ness is invisible to it. A DB-backed
approach (WatermelonDB / RxDB / Replicache) swaps only the persistence adapter + coordinator; the
VM still reads domain types.

> **Freshness.** `@tanstack/react-query-persist-client`, `query-async-storage-persister`, and
> `resumePausedMutations` are current TanStack v5; `@react-native-community/netinfo` is the
> maintained connectivity module. These library APIs evolve across majors — follow *their current
> docs*; the boundary (persist behind the cache, sync in a coordinator) is the durable part.

---

## 4. GraphQL — the client **is** the server-state boundary; the VM depends on a neutral hook

**The rule.** A GraphQL client fuses HTTP **and** the server-state cache, so it occupies the
server-state boundary and a pure-GraphQL app has **no separate `services/` HTTP layer** — the
client is it (the documented exception in [`stack-choices.md`](stack-choices.md)). The rule is
otherwise unchanged: cage the client + the documents in `queries/`, and expose the **same neutral
`ProductsData` shape** as triad §5, so the VM and View never learn it's GraphQL.

```ts
// features/products/queries/useProductsData.ts (GraphQL) — Apollo caged here; VM sees the §5 neutral shape
import { useCallback, useMemo } from 'react';
import { gql, useQuery } from '@apollo/client'; // the ONLY place Apollo is named
import { transformProduct, type ProductDTO } from '../transformers/transformProduct';
import { toAppError } from '@/shared/api/toAppError';
import type { ProductsData } from './types'; // the SAME neutral shape as triad §5

const PRODUCTS = gql`query Products { products { id title price rating } }`;

export function useProductsData(): ProductsData {
  const { data, loading, error, refetch } = useQuery<{ products: ProductDTO[] }>(PRODUCTS);
  const items = useMemo(() => (data?.products ?? []).map(transformProduct), [data]); // wire→domain at the edge, same as §3
  const reload = useCallback(() => { void refetch(); }, [refetch]); // stable ref (the §5 rule) — an inline arrow would change identity each render
  return {
    items,
    isLoading: loading,
    isRefreshing: false,
    error: error ? toAppError(error) : null, // GraphQL/network error → the domain AppError union (§1)
    refetch: reload,
  };
}
```

`useProductsViewModel` (triad §6), the View (§7), and the Screen (§8) are **byte-for-byte
unchanged** — they consume `ProductsData`. Writes: wrap `useMutation(gql\`…\`)` in `mutations/`
and return the neutral `ToggleFavorite` shape (triad §17). **urql / Relay**: same boundary, swap
the import.

> **Freshness.** Apollo Client's `gql` / `useQuery` / `useMutation` hooks are stable across its
> current majors; if you pin a specific version, confirm error-shape and cache APIs against *its*
> docs. The neutral hook absorbs any of that — the VM never sees it.

---

## 5. Deep linking — the inbound URL is untrusted input, validated by a parser at the boundary

**The rule.** A deep link is just an externally-supplied **route param** — i.e. untrusted input.
It builds directly on typed route params ([`triad-example.md`](triad-example.md) §14): the
URL→screen mapping lives in the **navigation layer**, and inbound params are **validated by a
`parser` at the boundary** before any ViewModel trusts them. No new layer — the navigation facade
+ parser already own it.

```ts
// the URL→screen mapping lives in the navigation layer, never a VM/View:
//   expo-router → the file tree IS the linking config (typed routes);
//   React Navigation → a `linking` object passed to NavigationContainer.

// features/products/navigation.ts — the SAME validated param hook as §14 (the parser is the guard)
import { useLocalSearchParams } from 'expo-router';      // RN Navigation: read route.params instead
import { parseRouteId } from './parsers/parseRouteId';

export function useProductRouteId(): number | null {
  const { id } = useLocalSearchParams<{ id: string }>();
  return parseRouteId(id); // a malformed deep link → null → the VM's `error` variant (§14), never a crash
}
```

The VM depends on the neutral `number | null` — a hostile or stale link can't smuggle bad data
inward, because the parser narrows it at the edge. **Auth-gated links**: the decision to redirect
to login is a **shell transition** owned by the navigation facade / a coordinator
([`worked-examples.md`](worked-examples.md) §8b), not the target screen's View — the View still
only renders the branch the VM resolved.

> **Why it protects the contract.** External input crosses exactly one boundary (navigation),
> is validated by one pure layer (the parser), and reaches the VM already typed — the same
> input→value discipline as every form field.

---

## 6. Security — token lifecycle, secret storage, PII (behind the auth service / `AuthBridge`)

**The rule.** Security lives behind the **`AuthBridge` port** already worked in
[`worked-examples.md`](worked-examples.md) §8d — that *is* the security boundary. The VM sees only
**domain state** (`isAuthenticated` via a selector, an `AppError` on failure), never a raw token
or the secure-storage API. Completing the rules around §8d:

- **Token lifecycle.** Access token in the client store (or memory); **refresh token in a secure
  keystore** (Keychain / Keystore), read async — which is exactly why §8d's `getRefreshToken`
  returns a `Promise`. The response interceptor refreshes once on `401`, then calls
  `onAuthFailure` → the **logout coordinator** ([`worked-examples.md`](worked-examples.md) §8c).
- **Single-flight refresh.** N concurrent `401`s must trigger **one** refresh, not N — queue the
  in-flight refresh inside the interceptor/bridge (the one place that owns transport), never in a
  VM. (A naive per-request refresh is the leak to catch in review.)
- **Secrets never in the client store or plain async storage** — a secure keystore only. The
  client store holds the *access* token (short-lived); the refresh token and any secret stay in
  the keystore behind the persistence adapter.
- **PII never logged.** The reporter (§1) scrubs before `report`; a `formatter` never builds a
  log/analytics string out of PII. User-facing PII on screen is a ready value from the VM, like any
  other copy.
- **Native security capabilities** (cert pinning, jailbreak/root detection, biometric gates) are
  **native modules → a `services/` device adapter** behind an abstraction, like any other native
  API (triad §2 / `stack-choices.md`); the VM sees a typed result + a domain error, never the
  native module.

No new code is needed beyond §8d + §8c + §1 — security is the **composition** of boundaries you
already have, plus the single-flight-refresh discipline. If a token, a `SecureStore` call, or a
refresh race ever appears in a ViewModel or a View, a boundary is leaking; the fix is to push it
back behind the `AuthBridge` / the device adapter.

---

## Where these sit on the rungs

Every recipe here is **rung-agnostic**, exactly like the triad: at screen-based they're top-level
`services/`, `coordinators/`, `queries/`, `navigation/`; at feature-based+ they move inside the
owning feature or `shared/` (the reporter, the persister, the sync coordinator, and the
`AuthBridge` are cross-cutting → `shared/`). None of them is a new *layer* — each is the existing
layer vocabulary (service · query · coordinator · parser · facade · persistence) applied to one
more capability. That is the whole point: the contract already had a home for all of this.
