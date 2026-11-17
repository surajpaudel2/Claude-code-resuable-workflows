# Frontend Coding Rules (Next.js, TypeScript)

## 1. Stack and project layout

Next.js (App Router) with TypeScript only, using a `src/` folder. The frontend mirrors the backend idea: feature folders play the role of modules, and `shared` holds what more than one feature uses.

### Folder structure

```
my-app/
├── src/
│   ├── app/                     routes only (Next.js decides this folder's shape)
│   │   ├── layout.tsx           shell around every page
│   │   ├── page.tsx             the "/" screen
│   │   ├── login/page.tsx       "/login"
│   │   ├── orders/
│   │   │   ├── page.tsx         "/orders"
│   │   │   └── [id]/page.tsx    "/orders/123"
│   │   └── payments/my/page.tsx "/payments/my"
│   ├── features/                one folder per business area
│   │   ├── auth/
│   │   ├── orders/
│   │   └── payments/
│   └── shared/                  api client, constants, utils, common UI
├── public/                      static files
├── .env.local                   real values, git-ignored
└── .env.example                 committed, every variable with a dummy value
```

### Rules

1. `app/` holds routes only. A folder inside `app/` is a URL segment, and `page.tsx` is the screen for it. `app/` itself is not part of the URL.
2. A `page.tsx` is thin. It reads the route parameters, calls code from a feature, and renders feature components. It holds no business logic and no API calls of its own.
3. `[id]/` is a dynamic segment. It matches any value, and Next.js passes the value to the page.
4. Each business area has one folder in `features/`. Pages that belong to the same area share it (`/orders` and `/orders/[id]` both use `features/orders/`).
5. Code used by two or more features lives in `shared/`, outside every feature.
6. Dependencies point one way: `app` uses `features`, `features` use `shared`. `shared` never imports from `features`, and `features` never import from `app`.
7. A feature never imports another feature's internals. Other features may use only what its `index.ts` exports (see section 2).
8. File and folder names are lowercase kebab-case (`get-order.api.ts`, `order-card.tsx`). Route folders follow the URL.
9. Secrets are never committed. Real values go in `.env.local`, and `.env.example` lists every variable with a dummy value. A variable without the `NEXT_PUBLIC_` prefix is visible to the server only, which is the default for everything here.

## 2. Feature structure and boundaries

The server's JSON shape must never reach the screens. Data passes through small steps, and each step is one folder. This is the same separation as the backend DTO, model and mapper.

### Folder structure (per feature)

```
features/<feature>/
├── api/            functions that call the backend, one per endpoint
├── dto/            raw shapes the backend sends and receives
├── mapper/         DTO <-> model conversion
├── model/          the shapes the UI uses
├── components/     screens, lists, forms
├── service/        use-case logic (added when a feature needs it)
└── index.ts        the feature's public exports
```

### Data flow

Backend JSON -> `api/` -> `dto/` -> `mapper/` -> `model/` -> `components/`

### Rules

1. Components only see models, never DTOs.
2. A DTO never leaves the `api/` folder. An `api/` function returns a model.
3. `dto/` types describe exactly what the backend sends. `model/` types describe what the UI wants (for example `createdAt` is a `Date`, not a string).
4. Mappers are the only code that knows both a DTO and a model. They convert and contain no business logic.
5. One mapper per source type and direction. Naming is `<Source>To<Target>`: file `order-response-dto-to-order.mapper.ts`, function `orderResponseDtoToOrder`.
6. If the backend renames or changes a field, only `dto/` and `mapper/` change.
7. A feature's `index.ts` exports only what other features may use: models, public components and public functions. Other features never import from `api/`, `dto/` or `mapper/` directly.
8. Types are declared with `type`. A `type` describes a shape only and carries no behaviour.

### Example

```ts
// features/orders/dto/order-response.dto.ts
export type OrderResponseDto = {
  id: string;
  status: "CREATED" | "SHIPPED" | "CANCELLED";
  amount: number;
  createdAt: string; // "2026-11-16T10:30:00Z"
};

// features/orders/model/order.model.ts
export type Order = {
  id: string;
  status: "CREATED" | "SHIPPED" | "CANCELLED";
  amount: number;
  createdAt: Date;
};

// features/orders/mapper/order-response-dto-to-order.mapper.ts
export function orderResponseDtoToOrder(dto: OrderResponseDto): Order {
  return {
    id: dto.id,
    status: dto.status,
    amount: dto.amount,
    createdAt: new Date(dto.createdAt),
  };
}
```

## 3. API layer

One file in `shared/api/` knows the backend's `ApiResponse<T>` envelope. Nothing else in the frontend does.

### Folder structure

```
shared/
├── constants/api-paths.ts      API_PATHS.V1 = "/api/v1"
└── api/
    ├── api-response.ts         ApiResponse<T>, ErrorInfo, FieldErrorInfo (mirror the backend)
    ├── api-error.ts            ApiError, the one error type for every failure
    └── api-client.ts           apiGet (and later apiPost, apiPut, apiPatch, apiDelete)

features/<feature>/api/
└── get-order.api.ts            one function per endpoint
```

### The envelope type

```ts
// shared/api/api-response.ts
export type ApiResponse<T> = {
  success: boolean;
  message: string;
  data?: T;              // "?" because the backend omits null fields
  pagination?: PageInfo;
  error?: ErrorInfo;
  correlationId: string;
  timestamp: string;
};
export type ErrorInfo = { code: string; details?: FieldErrorInfo[] };
export type FieldErrorInfo = { field: string; message: string };
```

### The error type

```ts
// shared/api/api-error.ts
export class ApiError extends Error {
  constructor(
    public readonly status: number,
    public readonly code: string,
    message: string,
    public readonly details: FieldErrorInfo[] = [],
    public readonly correlationId?: string,
  ) {
    super(message);
  }
}
```

### The client

```ts
// shared/api/api-client.ts
export async function apiGet<T>(path: string): Promise<T> {
  const res = await fetch(`${process.env.API_BASE_URL}${path}`, { cache: "no-store" });
  const body = (await res.json()) as ApiResponse<T>;

  if (!res.ok || !body.success) {
    throw new ApiError(res.status, body.error?.code ?? "UNKNOWN",
      body.message, body.error?.details, body.correlationId);
  }
  return body.data as T;
}
```

### A feature api function

```ts
// features/orders/api/get-order.api.ts
export async function getOrder(id: string): Promise<Order> {
  const dto = await apiGet<OrderResponseDto>(`${API_PATHS.V1}/orders/${id}`);
  return orderResponseDtoToOrder(dto);
}
```

### Rules

1. Only `shared/api/api-client.ts` reads the envelope. It returns the `data` part, typed by the caller (`apiGet<OrderResponseDto>`).
2. Every failure becomes an `ApiError` with the backend's stable `code`, the HTTP `status`, the field `details` and the `correlationId`. Screens switch on `code`, never on the message.
3. A feature api function is named `<verb><Resource>` (`getOrder`, `searchOrders`, `cancelOrder`). The file is named `<verb>-<resource>.api.ts`.
4. A feature api function takes plain values or models, calls the client, maps the DTO, and returns a model. It contains no other logic.
5. Paths use the `API_PATHS.V1` constant, never a repeated `/api/v1` string. The path rules are the backend's (plural kebab-case resources, no verbs, no trailing slash).
6. The base URL comes from the `API_BASE_URL` environment variable. It has no `NEXT_PUBLIC_` prefix, so it stays on the server. Settings are validated in one place (see the later configuration section).
7. `cache: "no-store"` is the default, so data is always fresh. Caching rules are added in a later section.
8. These functions run on the server first, because `page.tsx` is a Server Component by default. Calling them from the browser is covered in the later client components section.
9. An async function returns a `Promise<T>`. Callers use `await`.

## 4. Error handling

Expected failures (a missing order) are handled where they happen. Unexpected failures are thrown and caught by the nearest error boundary. This is the backend idea of module handlers first and a global handler as the fallback.

### Folder structure

```
app/
├── not-found.tsx            default 404 screen (also used for unmatched URLs)
├── error.tsx                default error screen
├── global-error.tsx         failures in the root layout
└── orders/
    └── error.tsx            optional, only if this area needs its own

shared/
├── api/not-found-on-404.ts  turns an ApiError 404 into notFound()
└── components/
    ├── error-screen.tsx     the one real error screen
    └── not-found-screen.tsx the one real 404 screen
```

### Rules

1. Two kinds of failure. Expected ones are handled explicitly in code. Unexpected ones are thrown and land in the nearest `error.tsx`.
2. Catch only what you can handle. A page never catches everything, the same as a backend module handler never catches `Exception`.
3. A missing resource is `ApiError` with status 404. Pages wrap the call with `notFoundOn404(...)`, which calls `notFound()` for a 404 and rethrows anything else.
4. `notFound()` works by throwing. Call it in the render path (a page, or a function the page awaits), never inside a promise that nobody awaits.
5. `error.tsx` must be a Client Component, so it starts with `"use client"`. The nearest `error.tsx` wins. Start with one default in `app/` and add a folder-level one only when that area needs different wording or behaviour.
6. Every `error.tsx` is thin. It renders the shared `ErrorScreen` and passes its props through. The real screen is written once, in `shared/components/`.
7. Errors thrown in Server Components reach `error.tsx` with a generic message and a `digest` reference, not the real details. In production the `ApiError` fields never reach the screen. The screen shows a safe generic text and the `digest` as a reference, and nothing else from the error.
8. The `error` prop is typed `unknown`. Narrow it (`instanceof Error`) before reading `digest`.
9. The real error is logged once, on the server, and the `digest` matches that log line. Error screens do not log or rethrow server errors again.
10. Use `retry()` for the "Try again" button. It re-fetches and re-renders the segment. Use `reset()` only to clear the error state without re-fetching. (`retry` is stable from Next.js 16.3. On an older version use `reset`.)
11. `global-error.tsx` replaces the root layout when it is active, so it must include its own `<html>` and `<body>` and cannot rely on global styles. Keep it minimal. It cannot export `metadata`.
12. The root `not-found.tsx` handles `notFound()` and any URL that matches no route. It is sent with every route, so it stays a Server Component with no interactive UI. `global-not-found` is experimental and is not used.
13. Error boundaries do not catch errors in event handlers or in async code that runs after rendering. Those are handled inside the component (see the client components section).
14. Not-authenticated (401) handling is defined with authentication, in a separate part.
15. Screen text is not hardcoded. It comes from message keys (see the messages and configuration section). The examples below use plain text only to stay short.

### The helper and a page

```ts
// shared/api/not-found-on-404.ts
import { notFound } from "next/navigation";

export async function notFoundOn404<T>(call: Promise<T>): Promise<T> {
  try {
    return await call;
  } catch (e) {
    if (e instanceof ApiError && e.status === 404) notFound(); // expected: show the 404 screen
    throw e;                                                   // anything else: bubble up
  }
}
```

```tsx
// app/orders/[id]/page.tsx
export default async function OrderPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const order = await notFoundOn404(getOrder(id));
  return <OrderDetails order={order} />;
}
```

### The screens

```tsx
// shared/components/error-screen.tsx
"use client";

export function ErrorScreen({ error, retry }: { error: unknown; retry: () => void }) {
  const digest = error instanceof Error ? (error as Error & { digest?: string }).digest : undefined;
  return (
    <div>
      <h2>Something went wrong</h2>
      {digest && <p>Reference: {digest}</p>}
      <button onClick={() => retry()}>Try again</button>
    </div>
  );
}
```

```tsx
// app/error.tsx
"use client";

export default function RootError(props: { error: unknown; retry: () => void }) {
  return <ErrorScreen {...props} />;
}
```

```tsx
// app/global-error.tsx
"use client";

export default function GlobalError({ retry }: { error: unknown; retry: () => void }) {
  return (
    <html>
      <body>
        <h2>Something went wrong</h2>
        <button onClick={() => retry()}>Try again</button>
      </body>
    </html>
  );
}
```

```tsx
// app/not-found.tsx  (a Server Component, so no "use client")
export default function NotFound() {
  return <NotFoundScreen />;
}
```
