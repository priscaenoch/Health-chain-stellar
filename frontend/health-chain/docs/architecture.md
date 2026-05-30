# Frontend Architecture & API Integration Guide

> **Audience:** New frontend contributors who understand React but are new to Stellar/blockchain.

---

## Table of Contents

1. [Tech Stack Overview](#tech-stack-overview)
2. [Folder Structure](#folder-structure)
3. [State Management](#state-management)
4. [API Service Layer](#api-service-layer)
5. [Stellar Integration](#stellar-integration)
6. [Role-Based Routing](#role-based-routing)
7. [Adding a New Page](#adding-a-new-page)
8. [Data-Flow Diagram](#data-flow-diagram)

---

## Tech Stack Overview

| Technology | Version | Purpose |
|---|---|---|
| [Next.js](https://nextjs.org/) | 16+ | React framework — App Router, SSR, middleware |
| [TypeScript](https://www.typescriptlang.org/) | 5+ | Static typing across the entire codebase |
| [Tailwind CSS](https://tailwindcss.com/) | 3+ | Utility-first styling |
| [shadcn/ui](https://ui.shadcn.com/) | — | Accessible, unstyled component primitives (via `components/ui/`) |
| [Zustand](https://zustand-demo.pmnd.rs/) | 5+ | Lightweight global state (auth, wallet) |
| [TanStack Query](https://tanstack.com/query) | 5+ | Server-state fetching, caching, and synchronization |
| [Stellar SDK / Freighter](https://developers.stellar.org/) | — | Wallet connection and transaction signing |
| [Socket.IO Client](https://socket.io/) | 4+ | Real-time WebSocket updates (order status, live ops) |
| [Lucide React](https://lucide.dev/) | — | Icon library |
| [Recharts](https://recharts.org/) | 3+ | Data visualization charts |
| [i18next / react-i18next](https://react.i18next.com/) | — | Internationalization |
| [Vitest](https://vitest.dev/) + [Testing Library](https://testing-library.com/) | — | Unit and component testing |

---

## Folder Structure

```
frontend/health-chain/
├── app/                        # Next.js App Router pages and layouts
│   ├── admin/                  # Admin-only pages (anomalies, riders, permissions…)
│   ├── auth/                   # Authentication pages (sign-in, sign-up)
│   ├── dashboard/              # Authenticated user dashboard pages
│   ├── onboarding/             # Partner onboarding wizard
│   ├── transparency/           # Public transparency dashboard
│   ├── layout.tsx              # Root layout (fonts, providers)
│   └── page.tsx                # Public landing page
│
├── components/                 # Reusable React components
│   ├── accessibility/          # ARIA helpers and focus management components
│   ├── admin/                  # Admin-specific UI components
│   ├── auth/                   # Sign-in / sign-up forms
│   ├── blockchain/             # Stellar contract activity feed
│   ├── dashboard/              # Dashboard widgets and charts
│   ├── providers/              # React context providers (ReactQuery, Toast, i18n)
│   ├── ui/                     # shadcn/ui primitives (Button, Card, Dialog…)
│   └── *.tsx                   # Shared layout components (Navbar, Footer…)
│
├── lib/                        # Non-UI logic
│   ├── api/                    # Typed API functions + HTTP client
│   │   ├── http-client.ts      # Core fetch wrapper with auth + retry
│   │   ├── queryKeys.ts        # Centralized TanStack Query key factory
│   │   └── *.api.ts            # One file per backend resource
│   ├── hooks/                  # Custom React hooks (data fetching, auth, UI)
│   ├── stores/                 # Zustand stores
│   │   └── auth.store.ts       # Auth tokens + user profile
│   ├── types/                  # Shared TypeScript types per domain
│   └── utils/                  # Pure utility functions
│       ├── cn.ts               # Tailwind class merging helper
│       ├── websocket-client.ts # Socket.IO wrapper for real-time updates
│       └── soroban-transaction-parser.ts  # Stellar transaction helpers
│
├── public/                     # Static assets (images, locale JSON files)
│   └── locales/                # i18n translation files
│
├── middleware.ts               # Next.js edge middleware (route protection)
├── next.config.ts              # Next.js configuration
├── tailwind.config.ts          # Tailwind theme and plugin config
└── tsconfig.json               # TypeScript compiler options
```

### Purpose of each `src/` directory at a glance

| Directory | What lives here |
|---|---|
| `app/` | Pages, layouts, and route segments (Next.js App Router) |
| `components/` | All React UI components, organized by feature |
| `lib/api/` | API call functions and the HTTP client |
| `lib/hooks/` | Custom hooks that combine API calls with React state |
| `lib/stores/` | Zustand global stores |
| `lib/types/` | TypeScript interfaces and enums shared across features |
| `lib/utils/` | Framework-agnostic helper functions |

---

## State Management

The app uses three distinct layers of state. Choosing the right layer keeps the codebase predictable.

### Zustand — global client state

Use Zustand for state that must persist across page navigations and is not derived from the server.

**Current stores:**

| Store | File | What it holds |
|---|---|---|
| Auth | `lib/stores/auth.store.ts` | `accessToken`, `refreshToken`, `user`, `isAuthenticated` |

The auth store uses Zustand's `persist` middleware backed by `sessionStorage` (cleared when the browser tab closes, reducing XSS exposure).

```typescript
// Reading auth state anywhere in the app
import { useAuthStore } from '@/lib/stores/auth.store';

const { user, isAuthenticated } = useAuthStore();

// Writing auth state (e.g., after login)
const { setTokens, setUser } = useAuthStore();
setTokens(access_token, refresh_token);
setUser(user);
```

**When to add a new Zustand store:** Only when the state is truly global, long-lived, and not a direct mirror of server data. Examples: wallet connection status, UI theme preference, notification badge count.

### TanStack Query — server state

All data fetched from the backend lives in the TanStack Query cache. This gives you automatic background refetching, loading/error states, and cache invalidation for free.

```typescript
// lib/hooks/useDashboardData.ts — typical pattern
import { useQuery } from '@tanstack/react-query';
import { fetchDashboardStats } from '@/lib/api/dashboard.api';
import { QUERY_KEYS } from '@/lib/api/queryKeys';

export function useDashboardStats() {
  return useQuery({
    queryKey: QUERY_KEYS.dashboard.stats(),
    queryFn: fetchDashboardStats,
  });
}
```

Mutations (POST/PATCH/DELETE) use `useMutation` and call `queryClient.invalidateQueries` on success to keep the cache fresh.

### Local component state

Use `useState` / `useReducer` for ephemeral UI state that does not need to be shared: form field values, modal open/close, accordion expanded state, etc.

---

## API Service Layer

### How it works

All HTTP calls go through `lib/api/http-client.ts`. The client:

- Automatically attaches the `Authorization: Bearer <token>` header from the Zustand auth store.
- Handles **401 Unauthorized** by refreshing the access token once, then retrying the original request.
- Queues concurrent requests during a token refresh so only one refresh call is made.
- Retries transient errors (5xx, 429) with full-jitter exponential backoff (up to 3 attempts).
- Redirects to `/auth/signin?reason=session_expired` if the refresh token is also invalid.

### Adding a new API call

1. **Create or open the relevant `*.api.ts` file** in `lib/api/`.

```typescript
// lib/api/blood-units.api.ts
import { api } from './http-client';
import type { BloodUnit } from '@/lib/types/blood-units';

const API_PREFIX = process.env.NEXT_PUBLIC_API_PREFIX || 'api/v1';

/** GET /api/v1/blood-units */
export async function fetchBloodUnits(): Promise<BloodUnit[]> {
  return api.get<BloodUnit[]>(`/${API_PREFIX}/blood-units`);
}

/** POST /api/v1/blood-units */
export async function createBloodUnit(data: Partial<BloodUnit>): Promise<BloodUnit> {
  return api.post<BloodUnit>(`/${API_PREFIX}/blood-units`, data);
}
```

2. **Add a query key** in `lib/api/queryKeys.ts` so TanStack Query can cache and invalidate correctly.

```typescript
// lib/api/queryKeys.ts (add to the existing object)
bloodUnits: {
  all: () => ['blood-units'] as const,
  detail: (id: string) => ['blood-units', id] as const,
},
```

3. **Create a hook** in `lib/hooks/` that wraps the API call with `useQuery` or `useMutation`.

```typescript
// lib/hooks/useBloodUnits.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { fetchBloodUnits, createBloodUnit } from '@/lib/api/blood-units.api';
import { QUERY_KEYS } from '@/lib/api/queryKeys';

export function useBloodUnits() {
  return useQuery({
    queryKey: QUERY_KEYS.bloodUnits.all(),
    queryFn: fetchBloodUnits,
  });
}

export function useCreateBloodUnit() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: createBloodUnit,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: QUERY_KEYS.bloodUnits.all() });
    },
  });
}
```

4. **Use the hook in a component.**

```typescript
function BloodUnitList() {
  const { data, isLoading, isError } = useBloodUnits();

  if (isLoading) return <p>Loading…</p>;
  if (isError) return <p>Failed to load blood units.</p>;

  return <ul>{data?.map(u => <li key={u.id}>{u.bloodType}</li>)}</ul>;
}
```

### Auth header pattern

The HTTP client reads the token from the Zustand store automatically. You never need to pass the token manually. For public endpoints that do not require authentication, pass `{ skipAuth: true }`:

```typescript
const publicData = await api.get('/public/stats', { skipAuth: true });
```

---

## Stellar Integration

### How wallet connect works

Health Chain uses the [Freighter](https://www.freighter.app/) browser extension as the Stellar wallet. Freighter injects a `window.freighter` API that the app calls to:

1. Request access to the user's public key.
2. Sign XDR-encoded Stellar transactions.

The wallet address is collected during the onboarding wizard (`app/onboarding/page.tsx`, the `wallet` step) and stored as part of the organization profile on the backend.

### How to build and sign a transaction

Stellar transactions are built on the backend (or via the Soroban SDK) and returned to the frontend as base64-encoded XDR. The frontend then asks Freighter to sign them.

```typescript
// Simplified example — actual implementation lives in components/blockchain/
import freighter from '@stellar/freighter-api';

async function signAndSubmit(xdr: string): Promise<string> {
  // 1. Ask Freighter to sign the XDR
  const { signedXDR } = await freighter.signTransaction(xdr, {
    network: 'TESTNET',  // or 'PUBLIC' for mainnet
  });

  // 2. Submit the signed transaction to the backend
  const result = await api.post('/api/v1/blockchain/submit', { xdr: signedXDR });
  return result.txHash;
}
```

The `lib/utils/soroban-transaction-parser.ts` utility parses raw Soroban contract call objects into human-readable `TransactionAction` descriptions shown in the signing modal (`components/modals/TransactionSigningModal`).

### Testnet vs. mainnet config

The network is controlled by the environment and by the Freighter extension setting. In development, always use **Testnet**. The Stellar Horizon API endpoints differ:

| Network | Horizon URL |
|---|---|
| Testnet | `https://horizon-testnet.stellar.org` |
| Mainnet (Public) | `https://horizon.stellar.org` |

The backend handles Horizon communication. The frontend only needs to tell Freighter which network to sign for (`'TESTNET'` or `'PUBLIC'`).

---

## Role-Based Routing

### UserRole and protected routes

The `user.role` field returned from the login endpoint determines which parts of the app a user can access.

| Role | Accessible routes |
|---|---|
| `ADMIN` | `/admin/**`, `/dashboard/**` |
| `HOSPITAL` | `/dashboard/**` |
| `BLOOD_BANK` | `/dashboard/**` |
| `RIDER` | `/dashboard/track-riders`, `/dashboard/orders` |
| _(unauthenticated)_ | `/`, `/transparency`, `/auth/**` |

### How route protection works

**Server-side (edge middleware):** `middleware.ts` runs on every request before the page renders. It reads the auth state from the `auth-storage` cookie (written by Zustand's `persist` middleware) and redirects unauthenticated users away from protected routes.

```
/dashboard  →  not authenticated  →  redirect to /auth/signin?redirect=/dashboard
/auth/signin  →  already authenticated  →  redirect to /dashboard
/transparency  →  always public, never redirected
```

**Client-side:** Components can read `user.role` from the Zustand auth store to conditionally render UI elements or redirect programmatically.

```typescript
import { useAuthStore } from '@/lib/stores/auth.store';

const { user } = useAuthStore();

if (user?.role !== 'ADMIN') {
  return <p>Access denied.</p>;
}
```

---

## Adding a New Page

Here is a step-by-step walkthrough for adding a new authenticated page at `/dashboard/my-feature`.

### Step 1 — Create the page file

```
app/dashboard/my-feature/page.tsx
```

```typescript
// app/dashboard/my-feature/page.tsx
export default function MyFeaturePage() {
  return <div>My Feature</div>;
}
```

Next.js App Router automatically registers this as the route `/dashboard/my-feature`. Because it lives under `app/dashboard/`, it inherits the dashboard layout (`app/dashboard/layout.tsx`) which includes the sidebar and navigation.

### Step 2 — Define your types

```typescript
// lib/types/my-feature.ts
export interface MyFeatureItem {
  id: string;
  name: string;
  status: 'active' | 'inactive';
}
```

### Step 3 — Create the API function

```typescript
// lib/api/my-feature.api.ts
import { api } from './http-client';
import type { MyFeatureItem } from '@/lib/types/my-feature';

const API_PREFIX = process.env.NEXT_PUBLIC_API_PREFIX || 'api/v1';

export async function fetchMyFeatureItems(): Promise<MyFeatureItem[]> {
  return api.get<MyFeatureItem[]>(`/${API_PREFIX}/my-feature`);
}
```

### Step 4 — Add a query key

```typescript
// lib/api/queryKeys.ts — add inside the exported object
myFeature: {
  all: () => ['my-feature'] as const,
},
```

### Step 5 — Create the hook

```typescript
// lib/hooks/useMyFeature.ts
import { useQuery } from '@tanstack/react-query';
import { fetchMyFeatureItems } from '@/lib/api/my-feature.api';
import { QUERY_KEYS } from '@/lib/api/queryKeys';

export function useMyFeatureItems() {
  return useQuery({
    queryKey: QUERY_KEYS.myFeature.all(),
    queryFn: fetchMyFeatureItems,
  });
}
```

### Step 6 — Wire up data fetching in the page

```typescript
// app/dashboard/my-feature/page.tsx
'use client';

import { useMyFeatureItems } from '@/lib/hooks/useMyFeature';

export default function MyFeaturePage() {
  const { data, isLoading, isError } = useMyFeatureItems();

  if (isLoading) return <p>Loading…</p>;
  if (isError) return <p>Something went wrong. Please try again.</p>;

  return (
    <ul>
      {data?.map((item) => (
        <li key={item.id}>{item.name} — {item.status}</li>
      ))}
    </ul>
  );
}
```

### Step 7 — Add a navigation link (optional)

If the page should appear in the sidebar, add a link to the sidebar component in `components/dashboard/` following the existing pattern.

---

## Data-Flow Diagram

The diagram below shows how data moves between the user's browser, the backend API, and the Stellar network.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser (Next.js)                        │
│                                                                 │
│  ┌──────────────┐    ┌─────────────────┐    ┌───────────────┐  │
│  │  Zustand     │    │  TanStack Query │    │   Freighter   │  │
│  │  auth.store  │    │  cache          │    │   Extension   │  │
│  │              │    │                 │    │   (wallet)    │  │
│  │ accessToken  │    │ server data     │    │               │  │
│  │ refreshToken │    │ (blood units,   │    │ signs XDR     │  │
│  │ user.role    │    │  orders, etc.)  │    │ transactions  │  │
│  └──────┬───────┘    └────────┬────────┘    └───────┬───────┘  │
│         │                    │                      │          │
│         └──────────┬─────────┘                      │          │
│                    │                                │          │
│            ┌───────▼────────┐                       │          │
│            │  http-client   │                       │          │
│            │  (lib/api/)    │                       │          │
│            │                │                       │          │
│            │ • Auth header  │                       │          │
│            │ • Token refresh│                       │          │
│            │ • Retry logic  │                       │          │
│            └───────┬────────┘                       │          │
└────────────────────┼───────────────────────────────-┼──────────┘
                     │ REST / WebSocket                │ signed XDR
                     ▼                                 ▼
          ┌──────────────────────┐         ┌──────────────────────┐
          │   Backend (NestJS)   │         │   Stellar Network    │
          │                      │         │                      │
          │  • Auth endpoints    │────────▶│  • Soroban contracts │
          │  • Blood unit CRUD   │ submit  │  • Ledger records    │
          │  • Order management  │  tx     │  • Token transfers   │
          │  • WebSocket gateway │         │  • Event indexing    │
          │  • Horizon proxy     │◀────────│                      │
          └──────────────────────┘  events └──────────────────────┘
```

**Flow summary:**

1. **User action** triggers a React component (button click, form submit, page load).
2. The component calls a **custom hook** (`lib/hooks/`).
3. The hook calls a **TanStack Query** `useQuery` or `useMutation`.
4. TanStack Query calls the **API function** (`lib/api/*.api.ts`).
5. The API function calls the **HTTP client** (`lib/api/http-client.ts`).
6. The HTTP client attaches the Bearer token from **Zustand** and sends the request to the **backend**.
7. For blockchain actions, the backend returns an XDR transaction. The frontend passes it to **Freighter** for signing, then submits the signed XDR back to the backend.
8. The backend submits the transaction to the **Stellar network** via Horizon and returns the result.
9. TanStack Query updates the cache and re-renders the component with fresh data.
