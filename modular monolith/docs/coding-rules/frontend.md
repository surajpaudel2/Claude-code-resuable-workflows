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

## 5. Forms and validation

Validation happens in three checks, like the backend validation levels. The backend is always the authority. The earlier checks only give faster feedback.

| Check | Where | What it checks | Result |
|---|---|---|---|
| 1. Browser | HTML attributes on the inputs (`required`, `min`, `maxLength`) | required, range, length | the browser's own message, no request sent |
| 2. Server action | Zod schema, `safeParse` | required, format, range | `FormState` with `fieldErrors` |
| 3. Backend | Bean Validation and business rules | shape, state, rules | 400 with field details, or 409 and 422 with an error code |

A form submits to a server action (a function marked `"use server"`). The browser never calls the backend itself, so `API_BASE_URL` and tokens stay on the server.

### Folder structure

```
shared/
├── forms/
│   ├── form-state.ts            FormState, the result every form action returns
│   ├── form-values-from.ts      FormData -> plain object of text values
│   └── field-errors-from.ts     backend FieldErrorInfo[] -> field -> messages
└── components/
    └── field-errors.tsx         shows the error list under one input

features/<feature>/
├── schema/
│   ├── create-order.schema.ts           Zod schema and the type inferred from it
│   └── create-order-form.messages.ts    message key constants
├── actions/
│   └── create-order.action.ts           the server action ("use server")
├── mapper/
│   └── create-order-form-to-create-order-request-dto.mapper.ts
├── api/create-order.api.ts              POST to the backend, returns a model
└── components/
    └── create-order-form.tsx            the form ("use client")
```

### Rules

1. The backend is the authority. Checks 1 and 2 must never be the only protection. Their values (`min`, `maxLength`, the Zod rules) are kept in step with the backend annotations.
2. Each form has its own Zod schema in `schema/`. Create and update forms have separate schemas, with no shared groups, the same as separate request classes in the backend.
3. The form type comes from `z.infer<typeof schema>`. It is never written by hand a second time.
4. Browser forms send text. Use `z.coerce` for numbers and dates.
5. Schemas hold message keys, not text. Keys are constants in a `<Form>FormMessages` object (`order.form.amount.positive`). The message service turns keys into text before they are returned in `FormState` (see the messages and configuration section). The examples below show the raw flow only.
6. A form submits to a server action in `actions/`. The file starts with `"use server"`. Actions are named `<verb><Resource>Action`.
7. Expected failures are returned as a `FormState`, not thrown. Unexpected failures are thrown and land in the nearest `error.tsx` (section 4).
8. An action does these steps in order: read the values, validate with the schema (check 2), call the api function (check 3), map errors, redirect.
9. A backend 400 uses `fieldErrorsFrom(e.details)` and fills `fieldErrors`. A 409 or 422 fills `formError` with the backend message. Any other `ApiError` is rethrown.
10. `redirect()` works by throwing. It is called outside every `try/catch`.
11. Form values go through a mapper to a request DTO (`<Form>FormTo<Request>Dto`), and the api function posts that DTO and returns a model. The action never builds a DTO by hand and never calls `fetch`.
12. React clears uncontrolled fields after an action finishes. Every error return includes `values`, and each input uses `defaultValue={state.values?.<field>}`. Sensitive fields (a password, a card number) are never put in `values`.
13. The form component is a Client Component. It uses `useActionState(action, initialState)`, passes `formAction` to `<form action>`, and sets `disabled={pending}` on the submit button.
14. Each input's `name` matches its schema field. Each input has a label linked with `htmlFor` and `id`.
15. Field errors are shown with the shared `FieldErrors` component. The form-level message uses `role="alert"`.
16. Check 1 uses plain HTML attributes. A browser-side Zod check needs an `onSubmit` handler that starts the action by hand (`startTransition`). That gives up React's automatic form reset and submission without JavaScript, so it is added only when a real need appears.
17. The page that shows the form stays a thin Server Component that renders the form component.
18. Labels and button text are not hardcoded. They use message keys. The examples use plain text only to stay short.
19. Tests: each schema constraint, and each action path (invalid input, a backend 400, a backend 409 or 422, success with redirect).

### The schema and its message keys

```ts
// features/orders/schema/create-order.schema.ts
import { z } from "zod";
import { CreateOrderFormMessages as M } from "./create-order-form.messages";

export const createOrderSchema = z.object({
  amount: z.coerce.number().positive(M.AMOUNT_POSITIVE),
  note: z.string().max(200, M.NOTE_MAX).optional(),
});

export type CreateOrderForm = z.infer<typeof createOrderSchema>;
```

```ts
// features/orders/schema/create-order-form.messages.ts
export const CreateOrderFormMessages = {
  AMOUNT_POSITIVE: "order.form.amount.positive",
  NOTE_MAX: "order.form.note.max",
} as const;
```

### Shared form types and helpers

```ts
// shared/forms/form-state.ts
export type FormState = {
  status: "idle" | "success" | "error";
  fieldErrors?: Record<string, string[]>; // field name -> text to show
  formError?: string;                     // one message for the whole form
  values?: Record<string, string>;        // what the user typed
};
```

```ts
// shared/forms/form-values-from.ts
export function formValuesFrom(formData: FormData): Record<string, string> {
  const values: Record<string, string> = {};
  for (const [key, value] of formData.entries()) {
    if (typeof value === "string") values[key] = value;
  }
  return values;
}
```

```ts
// shared/forms/field-errors-from.ts
export function fieldErrorsFrom(details: FieldErrorInfo[]): Record<string, string[]> {
  const result: Record<string, string[]> = {};
  for (const { field, message } of details) {
    (result[field] ??= []).push(message);
  }
  return result;
}
```

```tsx
// shared/components/field-errors.tsx
export function FieldErrors({ errors }: { errors?: string[] }) {
  if (!errors?.length) return null;
  return (
    <ul role="alert">
      {errors.map((e) => <li key={e}>{e}</li>)}
    </ul>
  );
}
```

### The server action

```ts
// features/orders/actions/create-order.action.ts
"use server";

export async function createOrderAction(_prev: FormState, formData: FormData): Promise<FormState> {
  const values = formValuesFrom(formData);
  const parsed = createOrderSchema.safeParse(values);                 // check 2
  if (!parsed.success) {
    return { status: "error", fieldErrors: z.flattenError(parsed.error).fieldErrors, values };
  }

  let order: Order;
  try {
    order = await createOrder(createOrderFormToCreateOrderRequestDto(parsed.data)); // check 3 runs in the backend
  } catch (e) {
    if (e instanceof ApiError) {
      if (e.status === 400) return { status: "error", fieldErrors: fieldErrorsFrom(e.details), values };
      if (e.status === 409 || e.status === 422) return { status: "error", formError: e.message, values };
    }
    throw e; // anything else goes to error.tsx
  }

  redirect(`/orders/${order.id}`); // outside the try/catch
}
```

### The form

```tsx
// features/orders/components/create-order-form.tsx
"use client";

const initialState: FormState = { status: "idle" };

export function CreateOrderForm() {
  const [state, formAction, pending] = useActionState(createOrderAction, initialState);

  return (
    <form action={formAction}>
      <label htmlFor="amount">Amount</label>
      <input id="amount" name="amount" type="number" step="0.01" min="0.01" required
             defaultValue={state.values?.amount} />
      <FieldErrors errors={state.fieldErrors?.amount} />

      <label htmlFor="note">Note</label>
      <textarea id="note" name="note" maxLength={200} defaultValue={state.values?.note} />
      <FieldErrors errors={state.fieldErrors?.note} />

      {state.formError && <p role="alert">{state.formError}</p>}
      <button disabled={pending}>Create order</button>
    </form>
  );
}
```

## 6. Pagination, filtering and sorting

The state of a list (page, size, sort, filters) lives in the URL, not in component state. The page reads the URL, calls the backend with the same parameters, and shows the result. Every control (next page, sort, filter) is a link or a plain GET form that changes the URL. This matches the backend list endpoints, which take `page`, `size`, `sort` and filters as query parameters.

### Folder structure

```
shared/
├── constants/page-defaults.ts       DEFAULT_PAGE=0, DEFAULT_SIZE=20, MAX_SIZE=100, DEFAULT_SORT="createdAt,desc"
├── api/
│   ├── api-response.ts              PageInfo (mirrors the backend)
│   └── api-client.ts                apiGetPage<T> and Paged<T>, next to apiGet
└── pagination/
    ├── page-params.schema.ts        pageField, sizeField, sortField(allowed)
    ├── page-result.ts               PageResult<T> and toPageResult
    ├── href-with.ts                 builds a link that changes some parameters
    └── next-sort.ts                 flips the sort direction

features/<feature>/
├── schema/order-search-params.schema.ts   ORDER_SORT_FIELDS and the search params schema
├── api/search-orders.api.ts               returns PageResult<Order>
└── components/
    ├── order-list.tsx                     table, sort links, Previous and Next
    └── order-filters.tsx                  the filter form
```

### Request flow

`/orders?page=1&size=20&sort=createdAt,desc&status=SHIPPED`

1. `page.tsx` awaits `searchParams` and validates it with the module's Zod schema. Bad or missing values become defaults.
2. `searchOrders(params)` builds the query string and calls the backend with `apiGetPage`.
3. `toPageResult` maps the DTOs to models and the `PageInfo` to a `PageResult`.
4. The page renders `OrderList` with the result and the current params.
5. A click on a link or a filter submit changes the URL, and the flow runs again.

### Rules

1. List state lives in the URL query string. No client state is used for it.
2. Parameter names and formats are the backend's: `page` is zero-based (default 0), `size` (default 20, max 100), `sort` is `field` or `field,asc|desc` with one field only. Filter names are camelCase.
3. The defaults and the size cap live in `shared/constants/page-defaults.ts`, and are kept in step with the backend `PageDefaults`.
4. A list page reads `searchParams`, which is a `Promise` and is awaited, and validates it with a Zod schema. A module builds its schema from the shared `pageField`, `sizeField` and `sortField` and adds its own filters.
5. A missing or invalid parameter falls back to its default with `.catch(...)`. A bad URL never shows an error screen, and `parse` never throws. This differs from forms, where invalid input is reported.
6. Each module defines its sort allowlist (`ORDER_SORT_FIELDS`), matching the backend `XSortFields` and the entity field names. Any other value falls back to the default sort.
7. Paged calls use `apiGetPage`, which returns the items and the `PageInfo`. `apiGet` returns only `data`. The envelope is still read in `api-client.ts` only.
8. A feature list function returns `PageResult<Model>` through `toPageResult(paged, mapper)`. Components never see DTOs, the envelope or `PageInfo`.
9. Only parameters that have a value are sent to the backend.
10. The list page and its components are Server Components. They have no state and no `"use client"`.
11. Previous and Next are `<Link>`s built with `hrefWith`. They appear only when `hasPrevious` or `hasNext` says so. The frontend never calculates this.
12. A sort link uses `nextSort` and resets `page` to 0. A new filter also goes back to the first page, because the form sends no `page`.
13. Filters are a `<form method="get">` that points at the list path. It carries `size` and `sort` as hidden inputs, and each field's `defaultValue` comes from the current params.
14. The URL page is zero-based, like the backend. Only the label shown to the user is one-based.
15. A page past the end returns an empty list with 200, so the screen shows an empty-state text.
16. Totals and page counts come from the backend. The frontend never counts.
17. Dates and money are formatted with an explicit locale and time zone (defined in a later section). The examples below show them raw to stay short.
18. Screen text uses message keys. The examples use plain text to stay short.
19. Tests: each schema fallback (bad page, size over the cap, unknown sort, bad filter), `toPageResult`, `hrefWith` and `nextSort`.

### Defaults and shared building blocks

```ts
// shared/constants/page-defaults.ts
export const PageDefaults = {
  DEFAULT_PAGE: 0,
  DEFAULT_SIZE: 20,
  MAX_SIZE: 100,
  DEFAULT_SORT: "createdAt,desc",
} as const;
```

```ts
// shared/pagination/page-params.schema.ts
export const pageField = z.coerce.number().int().min(0)
  .catch(PageDefaults.DEFAULT_PAGE);

export const sizeField = z.coerce.number().int().min(1).max(PageDefaults.MAX_SIZE)
  .catch(PageDefaults.DEFAULT_SIZE);

export function sortField(allowed: readonly string[]) {
  return z.string()
    .refine((s) => /^[A-Za-z]+(,(asc|desc))?$/.test(s) && allowed.includes(s.split(",")[0]))
    .catch(PageDefaults.DEFAULT_SORT);
}
```

### Module search params and the page

```ts
// features/orders/schema/order-search-params.schema.ts
export const ORDER_SORT_FIELDS = ["createdAt", "amount", "status"] as const;

export const orderSearchParamsSchema = z.object({
  page: pageField,
  size: sizeField,
  sort: sortField(ORDER_SORT_FIELDS),
  status: z.enum(["CREATED", "SHIPPED", "CANCELLED"]).optional().catch(undefined),
});

export type OrderSearchParams = z.infer<typeof orderSearchParamsSchema>;
```

```tsx
// app/orders/page.tsx
export default async function OrdersPage({
  searchParams,
}: {
  searchParams: Promise<Record<string, string | string[] | undefined>>;
}) {
  const params = orderSearchParamsSchema.parse(await searchParams);
  const result = await searchOrders(params);
  return (
    <>
      <OrderFilters params={params} />
      <OrderList result={result} params={params} />
    </>
  );
}
```

### The paged api call and the result model

```ts
// shared/api/api-response.ts
export type PageInfo = {
  page: number; size: number; totalElements: number;
  totalPages: number; hasNext: boolean; hasPrevious: boolean;
};
```

```ts
// shared/api/api-client.ts
export type Paged<T> = { items: T[]; pagination: PageInfo };

export async function apiGetPage<T>(path: string): Promise<Paged<T>> {
  const body = await send<T[]>(path); // the same fetch and ApiError handling as apiGet
  return { items: body.data ?? [], pagination: body.pagination as PageInfo };
}
```

`send` is the private fetch-and-error part of `apiGet`. `apiGet` returns `(await send<T>(path)).data as T`.

```ts
// shared/pagination/page-result.ts
export type PageResult<T> = {
  items: T[]; page: number; size: number;
  totalElements: number; totalPages: number; hasNext: boolean; hasPrevious: boolean;
};

export function toPageResult<D, M>(paged: Paged<D>, map: (dto: D) => M): PageResult<M> {
  const p = paged.pagination;
  return {
    items: paged.items.map(map),
    page: p.page, size: p.size, totalElements: p.totalElements,
    totalPages: p.totalPages, hasNext: p.hasNext, hasPrevious: p.hasPrevious,
  };
}
```

```ts
// features/orders/api/search-orders.api.ts
export async function searchOrders(params: OrderSearchParams): Promise<PageResult<Order>> {
  const query = new URLSearchParams({
    page: String(params.page), size: String(params.size), sort: params.sort,
  });
  if (params.status) query.set("status", params.status);

  const paged = await apiGetPage<OrderResponseDto>(`${API_PATHS.V1}/orders?${query}`);
  return toPageResult(paged, orderResponseDtoToOrder);
}
```

### Link helpers

```ts
// shared/pagination/href-with.ts
export function hrefWith(
  basePath: string,
  params: Record<string, string | number | undefined>,
  changes: Record<string, string | number | undefined>,
): string {
  const query = new URLSearchParams();
  for (const [key, value] of Object.entries({ ...params, ...changes })) {
    if (value !== undefined) query.set(key, String(value));
  }
  return `${basePath}?${query}`;
}
```

```ts
// shared/pagination/next-sort.ts
export function nextSort(current: string, field: string): string {
  const [currentField, direction = "asc"] = current.split(",");
  return `${field},${currentField === field && direction === "asc" ? "desc" : "asc"}`;
}
```

### The list and the filters

```tsx
// features/orders/components/order-list.tsx
import Link from "next/link";

export function OrderList({ result, params }: { result: PageResult<Order>; params: OrderSearchParams }) {
  return (
    <>
      <table>
        <thead>
          <tr>
            <th>Order</th>
            <th>Status</th>
            <th>
              <Link href={hrefWith("/orders", params, { sort: nextSort(params.sort, "amount"), page: 0 })}>Amount</Link>
            </th>
            <th>
              <Link href={hrefWith("/orders", params, { sort: nextSort(params.sort, "createdAt"), page: 0 })}>Created</Link>
            </th>
          </tr>
        </thead>
        <tbody>
          {result.items.map((order) => (
            <tr key={order.id}>
              <td><Link href={`/orders/${order.id}`}>{order.id}</Link></td>
              <td>{order.status}</td>
              <td>{order.amount}</td>
              <td>{order.createdAt.toLocaleDateString()}</td>
            </tr>
          ))}
        </tbody>
      </table>
      {result.items.length === 0 && <p>No orders found</p>}

      <nav aria-label="Pagination">
        {result.hasPrevious && (
          <Link href={hrefWith("/orders", params, { page: result.page - 1 })}>Previous</Link>
        )}
        <span>Page {result.page + 1} of {result.totalPages}</span>
        {result.hasNext && (
          <Link href={hrefWith("/orders", params, { page: result.page + 1 })}>Next</Link>
        )}
      </nav>
    </>
  );
}
```

```tsx
// features/orders/components/order-filters.tsx
export function OrderFilters({ params }: { params: OrderSearchParams }) {
  return (
    <form action="/orders" method="get">
      <label htmlFor="status">Status</label>
      <select id="status" name="status" defaultValue={params.status ?? ""}>
        <option value="">All</option>
        <option value="CREATED">Created</option>
        <option value="SHIPPED">Shipped</option>
        <option value="CANCELLED">Cancelled</option>
      </select>
      <input type="hidden" name="size" value={params.size} />
      <input type="hidden" name="sort" value={params.sort} />
      <button>Apply</button>
    </form>
  );
}
```

## 7. Client components

Components are Server Components by default. A Client Component is used only when the screen must react to the user (a click, state, a browser-only API). The browser never calls the backend directly. A Client Component calls a server action, and the action calls the api function. The server side keeps `API_BASE_URL`, tokens and the `ApiResponse` handling, and there is no CORS setup.

### Folder structure

```
shared/
├── actions/
│   └── action-result.ts          ActionResult, the result of a non-form action
└── api/
    └── api-client.ts             starts with import "server-only"; apiGet, apiGetPage, apiPost

features/<feature>/
├── actions/
│   └── cancel-order.action.ts    "use server"
├── api/
│   └── cancel-order.api.ts       POST to the backend, returns a model
└── components/
    └── cancel-order-button.tsx   "use client"
```

### Rules

1. Every component is a Server Component unless it needs the browser. Do not add `"use client"` "just in case".
2. Use a Client Component only for event handlers, state or effects (hooks), and browser-only APIs. Fetching data, layout, text and links stay on the server.
3. `"use client"` is the first line of the file, before the imports. The file and everything it imports are sent to the browser.
4. Keep Client Components small and at the leaves of the tree: a button, a form, a toggle. A page, a layout or a whole list is never a Client Component.
5. Props from a Server Component to a Client Component must be serializable data (strings, numbers, booleans, plain objects, arrays). Functions such as event handlers cannot cross. Pass only what the component needs (an `orderId`, not the whole order) and never secrets or tokens.
6. A Server Component can appear inside a Client Component only as `children` (or another prop). It is never imported by the Client Component.
7. The browser never calls the Spring backend. A Client Component calls a server action, and the action calls a feature api function.
8. A server action can be reached by a direct POST request, not only through the UI. Every action verifies authentication and authorization (see the authentication part), and the backend stays the authority.
9. A server action file starts with `"use server"` and exports only async functions, named `<verb><Resource>Action`. Expected failures are returned as an `ActionResult` or a `FormState`. Unexpected failures are thrown.
10. For a button or another event, use `useTransition` with `startTransition(async () => ...)` for the pending state, and `useState` for the error text. For a form submit, use the form pattern from section 5.
11. Do not hide unexpected errors with a `try/catch` in a handler. An unhandled error inside `startTransition` goes up to the nearest `error.tsx` (section 4). Error boundaries do not catch errors thrown directly in an event handler, so the handler returns expected failures from the action and lets the rest go through `startTransition`.
12. After a change, the action calls `revalidatePath(path)` so the affected page shows fresh data. `revalidatePath` runs only on the server, never in a Client Component.
13. `import "server-only"` goes at the top of `shared/api/api-client.ts` and of any module that reads private environment variables or secrets. A Client Component that imports such a module, directly or through other modules, fails the build. Modules that import `api-client` need no extra line. A `"use server"` action file is safe to import from a Client Component.
14. The backend URL and secrets never use the `NEXT_PUBLIC_` prefix.
15. A control is disabled while its action is `pending`.
16. Screen text uses message keys. The examples use plain text to stay short.
17. Tests: each action path (success, a returned 409 or 422 message, an unexpected error that is rethrown) and each component state (idle, pending, error shown).

### The server-only guard and the POST helper

```ts
// shared/api/api-client.ts
import "server-only";

export async function apiPost<T>(path: string, payload?: unknown): Promise<T> {
  const body = await send<T>(path, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: payload === undefined ? undefined : JSON.stringify(payload),
  });
  return body.data as T;
}
```

`send(path, init?)` is the private fetch-and-error function from section 6. It takes optional fetch settings, and `apiGet`, `apiGetPage` and `apiPost` all use it. Writes (`apiPost`, `apiPut`, `apiPatch`, `apiDelete`) follow the same pattern, so every failure is still an `ApiError` from one place.

### The api function, the action and the button

```ts
// features/orders/api/cancel-order.api.ts
export async function cancelOrder(id: string): Promise<Order> {
  const dto = await apiPost<OrderResponseDto>(`${API_PATHS.V1}/orders/${id}/cancel`);
  return orderResponseDtoToOrder(dto);
}
```

```ts
// shared/actions/action-result.ts
export type ActionResult =
  | { status: "success" }
  | { status: "error"; message: string };
```

```ts
// features/orders/actions/cancel-order.action.ts
"use server";

export async function cancelOrderAction(orderId: string): Promise<ActionResult> {
  try {
    await cancelOrder(orderId);
  } catch (e) {
    if (e instanceof ApiError && (e.status === 409 || e.status === 422)) {
      return { status: "error", message: e.message }; // e.g. ORDER_ALREADY_SHIPPED
    }
    throw e; // anything else goes to error.tsx
  }
  revalidatePath(`/orders/${orderId}`);
  return { status: "success" };
}
```

```tsx
// features/orders/components/cancel-order-button.tsx
"use client";

export function CancelOrderButton({ orderId }: { orderId: string }) {
  const [pending, startTransition] = useTransition();
  const [error, setError] = useState<string>();

  function handleClick() {
    startTransition(async () => {
      const result = await cancelOrderAction(orderId);
      setError(result.status === "error" ? result.message : undefined);
    });
  }

  return (
    <>
      <button onClick={handleClick} disabled={pending}>Cancel order</button>
      {error && <p role="alert">{error}</p>}
    </>
  );
}
```

The Server Component `OrderDetails` renders `<CancelOrderButton orderId={order.id} />`. The prop is a plain string, so it crosses the boundary.
