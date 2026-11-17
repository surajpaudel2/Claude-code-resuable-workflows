# Frontend Coding Rules (Next.js, TypeScript)

## Complete folder structure

One view of everything. Each folder is explained in the section named on its line. `orders` is the worked example feature, and every other feature has the same set of folders.

```
my-app/
├── .env.local                          real values, git-ignored (section 8)
├── .env.example                        committed, dummy values (section 8)
├── components.json                     shadcn CLI settings (section 12)
├── public/                             static files
└── src/
    ├── instrumentation.ts              startup check, calls getEnv() (section 8)
    │
    ├── app/                            ROUTES ONLY, thin pages (section 1)
    │   ├── globals.css                 imports Tailwind and shared/theme (section 11)
    │   ├── layout.tsx                  fonts, providers, <html suppressHydrationWarning>
    │   ├── page.tsx                    "/"
    │   ├── error.tsx                   default error screen (section 4)
    │   ├── global-error.tsx            root layout failures (section 4)
    │   ├── not-found.tsx               404 screen (section 4)
    │   ├── login/page.tsx              "/login"
    │   ├── orders/
    │   │   ├── page.tsx                "/orders" (list, section 6)
    │   │   ├── error.tsx               optional, only if needed
    │   │   └── [id]/page.tsx           "/orders/123"
    │   └── payments/my/page.tsx        "/payments/my"
    │
    ├── features/                       ONE FOLDER PER BUSINESS AREA (section 2)
    │   ├── auth/
    │   ├── payments/
    │   └── orders/
    │       ├── api/                    one function per endpoint (section 3)
    │       │   └── validation/         RESPONSE validation, one schema per endpoint (section 9)
    │       ├── constants/              order-status, order-limits, sort fields (section 10)
    │       ├── dto/                    raw backend shapes
    │       ├── mapper/                 DTO <-> model (<Source>To<Target>)
    │       ├── model/                  shapes the UI uses
    │       ├── schema/                 FIELD validation: Zod schemas, search params (sections 5, 6)
    │       ├── validation/
    │       │   └── business/           BUSINESS rules, one per file, plus the list (section 9)
    │       ├── actions/                server actions, "use server" (sections 5, 7)
    │       ├── messages/               orders.messages.ts, tOrders (section 8)
    │       ├── components/             one file when small, a folder when it has parts (section 13)
    │       ├── service/                reserved, added when needed
    │       └── index.ts                the only door other features may use
    │
    └── shared/                         USED BY 2+ FEATURES, never imports features
        ├── api/                        api-response, api-error, api-client, parse-response, not-found-on-404
        ├── actions/                    action-result
        ├── components/
        │   ├── ui/                     shadcn primitives (section 12)
        │   ├── buttons/                submit-button, icon-button, theme-toggle
        │   ├── forms/                  form-field, field-errors
        │   ├── feedback/               error-screen, not-found-screen, empty-state, badge, spinner
        │   ├── layout/                 page-header, container
        │   └── motion/                 fade-in, fade-up
        ├── config/                     env.ts, server only (section 8)
        ├── constants/                  routes, api-paths, api-error-codes, http-status, cookie-names, storage-keys, page-defaults (section 10)
        ├── forms/                      form-state, form-values-from, field-errors-from, translate-field-errors
        ├── format/                     format-config, format-date, format-money (section 8)
        ├── messages/                   create-translator, shared.messages (section 8)
        ├── pagination/                 page-params.schema, page-result, href-with, next-sort (section 6)
        ├── providers/                  app-providers: theme and motion config (section 11)
        ├── theme/                      tokens.css, typography.css, fonts, motion, theme-provider (section 11)
        ├── utils/                      cn
        └── validation/                 rule, runRules, field-errors-from-violations, fields/ (section 9)
```

Dependencies point one way: `app` uses `features`, `features` use `shared`, and never the reverse. Features talk to each other only through `index.ts`.

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
10. Styling uses Tailwind CSS with shadcn/ui components, `next-themes` for light and dark mode, and `motion` for animation (sections 11 and 12).
11. `@/` points to `src/` (the `create-next-app` default). Imports that cross a zone use `@/`. Imports inside one feature are relative.

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
  currency: string; // "AUD"
  createdAt: string; // "2026-11-16T10:30:00Z"
};

// features/orders/model/order.model.ts
export type Order = {
  id: string;
  status: "CREATED" | "SHIPPED" | "CANCELLED";
  amount: number;
  currency: string;
  createdAt: Date;
};

// features/orders/mapper/order-response-dto-to-order.mapper.ts
export function orderResponseDtoToOrder(dto: OrderResponseDto): Order {
  return {
    id: dto.id,
    status: dto.status,
    amount: dto.amount,
    currency: dto.currency,
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
export async function apiGet<T>(path: string, schema: z.ZodType<T>): Promise<T> {
  const res = await fetch(`${getEnv().API_BASE_URL}${path}`, { cache: "no-store" });
  const body = (await res.json()) as ApiResponse<unknown>;

  if (!res.ok || !body.success) {                       // envelope check
    throw new ApiError(res.status, body.error?.code ?? ApiErrorCodes.UNKNOWN,
      body.message, body.error?.details, body.correlationId);
  }
  return parseData(schema, body.data, body.correlationId); // shape check (section 9)
}
```

### A feature api function

```ts
// features/orders/api/get-order.api.ts
export async function getOrder(id: string): Promise<Order> {
  const dto = await apiGet(`${API_PATHS.V1}/orders/${id}`, orderResponseSchema);
  return orderResponseDtoToOrder(dto);
}
```

### Rules

1. Only `shared/api/api-client.ts` reads the envelope. It returns the `data` part after the shape check against the endpoint's schema (`apiGet(path, orderResponseSchema)`, section 9).
2. Every failure becomes an `ApiError` with the backend's stable `code`, the HTTP `status`, the field `details` and the `correlationId`. Screens switch on `code`, never on the message.
3. A feature api function is named `<verb><Resource>` (`getOrder`, `searchOrders`, `cancelOrder`). The file is named `<verb>-<resource>.api.ts`.
4. A feature api function takes plain values or models, calls the client, maps the DTO, and returns a model. It contains no other logic.
5. Paths use the `API_PATHS.V1` constant, never a repeated `/api/v1` string. The path rules are the backend's (plural kebab-case resources, no verbs, no trailing slash).
6. The base URL comes from the `API_BASE_URL` environment variable, read through `getEnv()` (section 8). It has no `NEXT_PUBLIC_` prefix, so it stays on the server.
7. `cache: "no-store"` is the default, so data is always fresh. Caching rules are added in a later section.
8. These functions run on the server only. A Client Component never calls them directly. It calls a server action (section 7).
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
└── components/feedback/
    ├── error-screen.tsx     the one real error screen
    └── not-found-screen.tsx the one real 404 screen
```

### Rules

1. Two kinds of failure. Expected ones are handled explicitly in code. Unexpected ones are thrown and land in the nearest `error.tsx`.
2. Catch only what you can handle. A page never catches everything, the same as a backend module handler never catches `Exception`.
3. A missing resource is `ApiError` with status 404. Pages wrap the call with `notFoundOn404(...)`, which calls `notFound()` for a 404 and rethrows anything else.
4. `notFound()` works by throwing. Call it in the render path (a page, or a function the page awaits), never inside a promise that nobody awaits.
5. `error.tsx` must be a Client Component, so it starts with `"use client"`. The nearest `error.tsx` wins. Start with one default in `app/` and add a folder-level one only when that area needs different wording or behaviour.
6. Every `error.tsx` is thin. It renders the shared `ErrorScreen` and passes its props through. The real screen is written once, in `shared/components/feedback/`.
7. Errors thrown in Server Components reach `error.tsx` with a generic message and a `digest` reference, not the real details. In production the `ApiError` fields never reach the screen. The screen shows a safe generic text and the `digest` as a reference, and nothing else from the error.
8. The `error` prop is typed `unknown`. Narrow it (`instanceof Error`) before reading `digest`.
9. The real error is logged once, on the server, and the `digest` matches that log line. Error screens do not log or rethrow server errors again.
10. Use `retry()` for the "Try again" button. It re-fetches and re-renders the segment. Use `reset()` only to clear the error state without re-fetching. (`retry` is stable from Next.js 16.3. On an older version use `reset`.)
11. `global-error.tsx` replaces the root layout when it is active, so it must include its own `<html>` and `<body>` and cannot rely on global styles. Keep it minimal. It cannot export `metadata`.
12. The root `not-found.tsx` handles `notFound()` and any URL that matches no route. It is sent with every route, so it stays a Server Component with no interactive UI. `global-not-found` is experimental and is not used.
13. Error boundaries do not catch errors in event handlers or in async code that runs after rendering. Those are handled inside the component (see the client components section).
14. Not-authenticated (401) handling is defined with authentication, in a separate part.
15. Screen text is not hardcoded. It comes from message keys (section 8). The examples below use plain text only to stay short.

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
// shared/components/feedback/error-screen.tsx
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
└── components/forms/
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
5. Schemas hold message keys, not text. Keys are constants in a `<Form>FormMessages` object (`order.form.amount.positive`), checked against the feature's message file (section 8). The action turns the keys into text with `translateFieldErrors` before it returns `FormState`. A message used by a schema has no placeholders, because a schema can carry only a key.
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
} as const satisfies Record<string, OrderMessageKey>;
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
// shared/components/forms/field-errors.tsx
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
    return {
      status: "error",
      fieldErrors: translateFieldErrors(z.flattenError(parsed.error).fieldErrors, orderMessages),
      values,
    };
  }

  const violations = runRules(createOrderRules, parsed.data);        // business rules (section 9)
  if (violations.length > 0) {
    return {
      status: "error",
      fieldErrors: fieldErrorsFromViolations(violations, orderMessages),
      values,
    };
  }

  let order: Order;
  try {
    order = await createOrder(createOrderFormToCreateOrderRequestDto(parsed.data)); // check 3 runs in the backend
  } catch (e) {
    return backendErrorToFormState(e, values); // section 13 shows this helper
  }

  redirect(ROUTES.ORDERS.DETAIL(order.id)); // outside the try/catch
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
17. Dates and money are shown with `formatDateTime` and `formatMoney` (section 8), never with `toLocaleDateString` or a raw number.
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

export async function apiGetPage<T>(path: string, itemSchema: z.ZodType<T>): Promise<Paged<T>> {
  const body = await send<unknown[]>(path); // the same fetch and envelope check as apiGet
  return {
    items: parseData(z.array(itemSchema), body.data ?? [], body.correlationId),
    pagination: body.pagination as PageInfo,
  };
}
```

`send` is the private fetch-and-error part of `apiGet`. `apiGet` is `send` followed by `parseData(schema, body.data, body.correlationId)`.

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

  const paged = await apiGetPage(`${API_PATHS.V1}/orders?${query}`, orderResponseSchema);
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
              <td>{formatMoney(order.amount, order.currency)}</td>
              <td>{formatDateTime(order.createdAt)}</td>
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

export async function apiPost<T>(path: string, schema: z.ZodType<T>, payload?: unknown): Promise<T> {
  const body = await send<unknown>(path, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: payload === undefined ? undefined : JSON.stringify(payload),
  });
  return parseData(schema, body.data, body.correlationId);
}
```

`send(path, init?)` is the private fetch-and-error function from section 6. It takes optional fetch settings, and `apiGet`, `apiGetPage` and `apiPost` all use it. Writes (`apiPost`, `apiPut`, `apiPatch`, `apiDelete`) follow the same pattern, so every failure is still an `ApiError` from one place.

### The api function, the action and the button

```ts
// features/orders/api/cancel-order.api.ts
export async function cancelOrder(id: string): Promise<Order> {
  const dto = await apiPost(`${API_PATHS.V1}/orders/${id}/cancel`, orderResponseSchema);
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

## 8. Messages, configuration and formatting

No text that we write is hardcoded. Code holds message keys, and one function turns a key into text. Settings are read once, checked against a schema, and the app refuses to start if one is missing. Dates and money are shown in one agreed format.

### Folder structure

```
shared/
├── messages/
│   ├── create-translator.ts        createTranslator(messages)
│   └── shared.messages.ts          shared texts and tShared
├── config/
│   └── env.ts                      the validated environment (server only)
├── forms/
│   └── translate-field-errors.ts   Zod keys -> text for FormState
└── format/
    ├── format-config.ts            FormatConfig.LOCALE and TIME_ZONE
    ├── format-date.ts              formatDateTime
    └── format-money.ts             formatMoney

features/<feature>/
├── messages/
│   └── orders.messages.ts          the feature's key -> text, and tOrders
└── schema/
    └── create-order-form.messages.ts   key constants used by the schema

src/instrumentation.ts              the startup check
.env.local, .env.example            project root, not inside src/
```

### Message rules

1. No text that we write is hardcoded in components, schemas or actions. Code holds keys only. Text that comes from the backend is already finished text and is shown as it is.
2. Each feature owns `messages/<feature>.messages.ts`: an `as const` object of key -> text, and its translator made with `createTranslator` (`tOrders`). Shared texts live in `shared/messages/shared.messages.ts` with `tShared`.
3. Keys are `<feature>.<area>.<name>`, lowercase with dots (`order.form.amount.positive`), the same naming as the backend. Shared keys start with `shared.`.
4. English only for now. No i18n library is added until a second language is needed. When it is, the keys stay and only the translator is replaced.
5. `t(key, ...args)` accepts only keys that exist in its file, so a typo fails the build. Dynamic values use `{0}`, `{1}` placeholders. Text is never concatenated.
6. Components call `t("literal.key")`. A constants object (`<Form>FormMessages`) is used only where a key is carried without calling `t`, for example in a schema. It is declared `as const satisfies Record<string, <Feature>MessageKey>`.
7. A feature uses only its own translator and `tShared`, never another feature's.
8. A message used by a schema has no placeholders, because a schema can carry only a key. The text contains the number itself and is kept in step with the schema.
9. Zod keys become text in the server action with `translateFieldErrors(fieldErrors, <feature>Messages)` before `FormState` is returned. Every constraint in a schema has a key. A message that is not one of our keys is replaced by `shared.form.invalid`.
10. Backend text (`ApiError.message`, the field `details`) is shown as it is and never looked up. Screens decide what to do from `ApiError.code`, never from the text.
11. Message files are not secret. They are used in both server and client components.

### Configuration rules

12. Only `shared/config/env.ts` reads `process.env`. Everything else calls `getEnv()`.
13. `env.ts` starts with `import "server-only"` and holds a Zod schema that lists every variable and its type. An invalid or missing value throws an error that lists the fields.
14. `src/instrumentation.ts` calls `getEnv()` inside `register()` (Node runtime only). Next.js completes `register()` before the server handles requests, so a bad configuration fails at startup and not on the first request.
15. Required settings have no default in the schema. Values that are only for local work go in `.env.local`.
16. `.env.local` (git-ignored, real values) and `.env.example` (committed, every variable with a dummy value) live in the project root, not inside `src/`. In production, environment variables set by the platform are used.
17. Variables are server-only by default. The `NEXT_PUBLIC_` prefix is used only for values that are safe for the browser, because they are inlined at build time and cannot change after the build. The backend URL and secrets never use it.
18. Variable names are `UPPER_SNAKE_CASE`.
19. Tests: the env schema (valid, missing, malformed), `createTranslator` (placeholders), and `translateFieldErrors` (a known key, an unknown message).

### Formatting rules

20. Time travels as UTC ISO strings, and the mapper turns them into `Date`. Nothing is shown with `toLocaleString` or `toLocaleDateString` without explicit settings.
21. Dates are shown with `formatDateTime` and money with `formatMoney`. Both use `Intl` formatters created once, with an explicit locale and time zone from `FormatConfig`.
22. Changing the display locale or time zone is one edit in `format-config.ts`.
23. Money is display-only. The frontend does no arithmetic on amounts, and the backend computes totals. A model that has an amount also carries its currency code.
24. Tests: `formatDateTime` and `formatMoney` with fixed inputs.

### Messages

```ts
// shared/messages/create-translator.ts
export function createTranslator<T extends Record<string, string>>(messages: T) {
  return function t(key: keyof T & string, ...args: (string | number)[]): string {
    const template = messages[key];
    return args.reduce<string>(
      (text, arg, i) => text.replaceAll(`{${i}}`, String(arg)),
      template,
    );
  };
}
```

```ts
// shared/messages/shared.messages.ts
export const sharedMessages = {
  "shared.form.invalid": "This value is not valid",
  "shared.api.invalid_response": "The server sent an unexpected response",
} as const;

export const tShared = createTranslator(sharedMessages);
```

```ts
// features/orders/messages/orders.messages.ts
export const orderMessages = {
  "order.form.amount.label": "Amount",
  "order.form.amount.positive": "Amount must be greater than 0",
  "order.form.note.max": "Note must be at most 200 characters",
  "order.form.submit": "Create order",
} as const;

export const tOrders = createTranslator(orderMessages);
export type OrderMessageKey = keyof typeof orderMessages;
```

```ts
// features/orders/schema/create-order-form.messages.ts
export const CreateOrderFormMessages = {
  AMOUNT_POSITIVE: "order.form.amount.positive",
  NOTE_MAX: "order.form.note.max",
} as const satisfies Record<string, OrderMessageKey>;
```

```tsx
// in a component
<label htmlFor="amount">{tOrders("order.form.amount.label")}</label>
<button disabled={pending}>{tOrders("order.form.submit")}</button>
```

```ts
// shared/forms/translate-field-errors.ts
export function translateFieldErrors(
  fieldErrors: Record<string, string[] | undefined>,
  messages: Record<string, string>,
): Record<string, string[]> {
  const result: Record<string, string[]> = {};
  for (const [field, keys] of Object.entries(fieldErrors)) {
    result[field] = (keys ?? []).map(
      (key) => messages[key] ?? tShared("shared.form.invalid"),
    );
  }
  return result;
}
```

### Configuration

```ts
// shared/config/env.ts
import "server-only";
import { z } from "zod";

const envSchema = z.object({
  API_BASE_URL: z.string().url(),
});

export type Env = z.infer<typeof envSchema>;

let cached: Env | undefined;

export function getEnv(): Env {
  if (cached) return cached;
  const parsed = envSchema.safeParse(process.env);
  if (!parsed.success) {
    throw new Error(
      `Invalid environment variables: ${JSON.stringify(z.flattenError(parsed.error).fieldErrors)}`,
    );
  }
  cached = parsed.data;
  return cached;
}
```

```ts
// src/instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    const { getEnv } = await import("./shared/config/env");
    getEnv(); // fails here if a variable is missing or invalid
  }
}
```

### Formatting

```ts
// shared/format/format-config.ts
export const FormatConfig = {
  LOCALE: "en",
  TIME_ZONE: "UTC",
} as const;
```

```ts
// shared/format/format-date.ts
const dateTime = new Intl.DateTimeFormat(FormatConfig.LOCALE, {
  dateStyle: "medium",
  timeStyle: "short",
  timeZone: FormatConfig.TIME_ZONE,
});

export function formatDateTime(date: Date): string {
  return dateTime.format(date);
}
```

```ts
// shared/format/format-money.ts
export function formatMoney(amount: number, currency: string): string {
  return new Intl.NumberFormat(FormatConfig.LOCALE, { style: "currency", currency }).format(amount);
}
```

```tsx
// in a list row
<td>{formatMoney(order.amount, order.currency)}</td>
<td>{formatDateTime(order.createdAt)}</td>
```

## 9. Validation

A value is checked in three different ways for three different reasons. Each kind has its own folder, so a reader always knows where a check lives.

| Kind | Question it answers | Lives in | Runs |
|---|---|---|---|
| Field | Is each value well-formed (type, length, format)? | `schema/`, reusable parts in `shared/validation/fields/` | In the action, before any backend call |
| Business | Is this combination of values allowed? | `validation/business/` | In the action, after field validation passes |
| Response | Did the backend send what it promised? | `api/validation/` | On every backend response |

### Folder structure

```
shared/
├── validation/
│   ├── rule.ts                          Rule<T>, RuleViolation, runRules
│   ├── field-errors-from-violations.ts  violations -> FormState field errors
│   └── fields/                          reusable field definitions (moneyField, emailField)
└── api/
    ├── parse-response.ts                parseData: the response shape check
    └── api-client.ts                    apiGet(path, schema), apiGetPage, apiPost

features/<feature>/
├── schema/                              FIELD validation (sections 5 and 6)
├── constants/order-limits.ts            the numbers the rules use
├── validation/
│   └── business/                        BUSINESS validation
│       ├── amount-within-limit.rule.ts           one rule per file
│       ├── note-required-for-large-order.rule.ts
│       ├── create-order-rules.ts                 the list of rules for the form
│       └── create-order-rule.messages.ts         message key constants
└── api/
    ├── get-order.api.ts
    └── validation/
        └── order-response.schema.ts     RESPONSE validation, one schema per endpoint
```

### How the three fit together

1. The browser form sends the values to the server action.
2. The action runs field validation (Zod). If it fails, it returns the field errors.
3. The action runs the business rules on the validated values. If any rule is violated, it returns all the violations as field errors.
4. The action calls the backend. The backend repeats every rule and is the final authority.
5. The api function checks the backend's response shape before it maps it.

### Rules

1. The three kinds stay apart. A schema never holds a business rule, a business rule never checks a format, and a response schema is never reused for a form.
2. The frontend has only business rules that need nothing but the submitted values and fixed limits (a comparison between two fields, a maximum amount). Anything that needs stored data (stock, balance, duplicates, permissions) is decided by the backend and shown through its field errors (section 5).
3. The backend repeats every rule. Frontend rules exist for quick feedback and are never the security.
4. A business rule is one function of the type `Rule<T>`. It takes the validated input and returns a `RuleViolation` or `null`. One rule per file, named `<what-it-checks>.rule.ts`.
5. A rule is a pure function. It does not throw, call the network, read the clock or change its input.
6. The rules for a form are listed in `<form>-rules.ts`. `runRules` runs every rule and collects all violations, so the user sees every problem at once. Adding a rule means one new file and one new line in the list. The runner, the action and the form do not change. This is the strategy pattern, and it keeps the code open for extension and closed for modification.
7. A violation carries a `field` and a message key, never text. Keys are constants in `<form>-rule.messages.ts`, checked against the feature's message file (section 8). Limits come from `constants/` (`OrderLimits`), never typed inside a rule.
8. An action runs the stages in order: field validation, business rules, backend call. The first stage that fails returns a `FormState` with `fieldErrors`, and the later stages do not run.
9. New reusable field definitions go in `shared/validation/fields/`. The page, size and sort fields stay in `shared/pagination/` (section 6).
10. Every client call takes a Zod schema: `apiGet(path, schema)`, `apiGetPage(path, itemSchema)`, `apiPost(path, schema, payload)`. The envelope is checked first, then the shape of `data`.
11. A failed shape check throws `ApiError` with status 502 and code `INVALID_RESPONSE`. The user sees the generic message, never the list of issues.
12. Each endpoint has one response schema in `api/validation/<name>-response.schema.ts`. It is declared with `satisfies z.ZodType<Dto>`, so the build fails when the schema and the DTO type drift apart. DTO types stay in `dto/`.
13. A response schema lists only the fields the frontend uses. Extra fields are ignored. If the backend renames or removes a field we use, the failure appears at the api boundary, not as "Invalid Date" on a screen.
14. Tests: each rule (violated and not violated), `runRules` collecting more than one violation, and each response schema (a valid sample passes, a missing field fails).

### The rule type and the runner

```ts
// shared/validation/rule.ts
export type RuleViolation = { field: string; key: string };

export type Rule<T> = (input: T) => RuleViolation | null;

export function runRules<T>(rules: readonly Rule<T>[], input: T): RuleViolation[] {
  return rules.flatMap((rule) => rule(input) ?? []);
}
```

```ts
// shared/validation/field-errors-from-violations.ts
export function fieldErrorsFromViolations(
  violations: RuleViolation[],
  messages: Record<string, string>,
): Record<string, string[]> {
  const keys: Record<string, string[]> = {};
  for (const { field, key } of violations) {
    (keys[field] ??= []).push(key);
  }
  return translateFieldErrors(keys, messages);
}
```

### Business rules for the create-order form

```ts
// features/orders/constants/order-limits.ts
export const OrderLimits = {
  MAX_AMOUNT: 10_000,
  NOTE_REQUIRED_ABOVE: 5_000,
} as const;
```

```ts
// features/orders/validation/business/create-order-rule.messages.ts
export const CreateOrderRuleMessages = {
  AMOUNT_TOO_HIGH: "order.rule.amount.max",
  NOTE_REQUIRED: "order.rule.note.required",
} as const satisfies Record<string, OrderMessageKey>;
```

```ts
// added to features/orders/messages/orders.messages.ts
"order.rule.amount.max": "Amount must be at most 10000",
"order.rule.note.required": "A note is required for orders over 5000",
```

The text contains the number itself, so it is kept in step with `OrderLimits` (section 8, rule 8).

```ts
// features/orders/validation/business/amount-within-limit.rule.ts
export const amountWithinLimit: Rule<CreateOrderForm> = (input) =>
  input.amount > OrderLimits.MAX_AMOUNT
    ? { field: "amount", key: CreateOrderRuleMessages.AMOUNT_TOO_HIGH }
    : null;
```

```ts
// features/orders/validation/business/note-required-for-large-order.rule.ts
export const noteRequiredForLargeOrder: Rule<CreateOrderForm> = (input) =>
  input.amount > OrderLimits.NOTE_REQUIRED_ABOVE && !input.note
    ? { field: "note", key: CreateOrderRuleMessages.NOTE_REQUIRED }
    : null;
```

```ts
// features/orders/validation/business/create-order-rules.ts
export const createOrderRules: readonly Rule<CreateOrderForm>[] = [
  amountWithinLimit,
  noteRequiredForLargeOrder,
];
```

The action (section 5) runs them after field validation and before the backend call:

```ts
const violations = runRules(createOrderRules, parsed.data);
if (violations.length > 0) {
  return {
    status: "error",
    fieldErrors: fieldErrorsFromViolations(violations, orderMessages),
    values,
  };
}
```

### Response validation

```ts
// features/orders/api/validation/order-response.schema.ts
import { z } from "zod";

export const orderResponseSchema = z.object({
  id: z.string(),
  amount: z.number(),
  currency: z.string(),
  createdAt: z.iso.datetime(),
}) satisfies z.ZodType<OrderResponseDto>;
```

```ts
// shared/api/parse-response.ts
export function parseData<T>(schema: z.ZodType<T>, data: unknown, correlationId?: string): T {
  const parsed = schema.safeParse(data);
  if (!parsed.success) {
    throw new ApiError(
      HttpStatus.BAD_GATEWAY,
      ApiErrorCodes.INVALID_RESPONSE,
      tShared("shared.api.invalid_response"),
      [],
      correlationId,
    );
  }
  return parsed.data;
}
```

`apiGet`, `apiGetPage` and `apiPost` call `parseData` after the envelope check (sections 3, 6 and 7 show the calls).

## 10. Constants

Code never holds a bare string or number that means something. Text for the user is a message key (section 8). Everything else is a named constant.

### Folder structure

```
shared/constants/
├── routes.ts              ROUTES, every URL of the app
├── api-paths.ts           API_PATHS.V1
├── api-error-codes.ts     ApiErrorCodes, the backend's stable codes we handle
├── http-status.ts         HttpStatus
├── cookie-names.ts        CookieNames
├── storage-keys.ts        StorageKeys
└── page-defaults.ts       page and size defaults (section 6)

features/<feature>/constants/
├── order-status.ts        OrderStatus
├── order-limits.ts        OrderLimits
└── order-sort-fields.ts   ORDER_SORT_FIELDS
```

### Rules

1. No magic strings or numbers. A value that means something (a URL, a status, a limit, a cookie name, an error code) is a named constant.
2. A constant that is used by two or more features lives in `shared/constants/`. A constant that belongs to one feature lives in that feature's `constants/`.
3. A group of constants is an `as const` object with `UPPER_SNAKE_CASE` keys (`OrderLimits.MAX_AMOUNT`), the same naming as the backend constants. A single constant is `UPPER_SNAKE_CASE` too.
4. A set of allowed values is an `as const` object, plus a type made from its values. We do not use TypeScript `enum`.
5. Links are never typed by hand. Use `ROUTES` (`ROUTES.ORDERS.DETAIL(order.id)`). A route that takes a parameter is a function in `ROUTES`.
6. Code compares `error.code` with `ApiErrorCodes` and `status` with `HttpStatus`, never with a literal. The earlier examples show literals only to stay short.
7. Limits, defaults and allowed values that the backend also knows are kept in step with the backend and say so in a comment.
8. Constants hold no text meant for the user. That is a message key.

### Examples

```ts
// shared/constants/routes.ts
export const ROUTES = {
  HOME: "/",
  LOGIN: "/login",
  ORDERS: {
    LIST: "/orders",
    NEW: "/orders/new",
    DETAIL: (id: string) => `/orders/${id}`,
  },
  PAYMENTS: { MY: "/payments/my" },
} as const;
```

```ts
// shared/constants/http-status.ts
export const HttpStatus = {
  BAD_REQUEST: 400,
  NOT_FOUND: 404,
  CONFLICT: 409,
  UNPROCESSABLE: 422,
  BAD_GATEWAY: 502,
} as const;
```

```ts
// shared/constants/api-error-codes.ts
export const ApiErrorCodes = {
  UNKNOWN: "UNKNOWN",
  INVALID_RESPONSE: "INVALID_RESPONSE",
} as const;
```

```ts
// features/orders/constants/order-status.ts
export const OrderStatus = {
  PENDING: "PENDING",
  PAID: "PAID",
  SHIPPED: "SHIPPED",
  CANCELLED: "CANCELLED",
} as const;

export type OrderStatus = (typeof OrderStatus)[keyof typeof OrderStatus];
```

```tsx
// a link, and a comparison
<Link href={ROUTES.ORDERS.DETAIL(order.id)}>...</Link>
if (order.status === OrderStatus.SHIPPED) { ... }
```

## 11. Styling, theme and motion

Styling uses Tailwind CSS v4 with shadcn/ui components. Colours, typography and motion are each defined in one place. Components use only those tokens, so a change to the look is one edit, and light and dark mode both work without extra code in the components.

### How it fits together

- Tailwind v4 is configured in CSS. There is no `tailwind.config.js`.
- shadcn/ui components read colours from CSS variables. The light values are in `:root` and the dark values in `.dark`. A block called `@theme inline` maps those variables to Tailwind classes such as `bg-background`.
- `next-themes` puts the `dark` class on the `<html>` element, so the variables switch with it. It follows the system setting by default and remembers the user's choice.
- `motion` animates. `MotionConfig` with `reducedMotion="user"` turns off transform and layout animations for users who ask for reduced motion, and keeps opacity and colour changes.

### Folder structure

```
src/app/
├── globals.css               imports Tailwind and the theme files
└── layout.tsx                fonts, providers, <html suppressHydrationWarning>

shared/theme/
├── tokens.css                colour variables: :root (light), .dark (dark), radius, @theme inline
├── typography.css            font families, type roles (text-title, text-body, ...)
├── fonts.ts                  next/font loaders
├── motion.ts                 durations, easings, springs, variants
└── theme-provider.tsx        wraps next-themes (Client Component)

shared/providers/
└── app-providers.tsx         ThemeProvider + MotionConfig, used once in layout.tsx

components.json               shadcn settings (paths and aliases)
```

### Rules

1. Colours are defined only in `tokens.css`, as hex values, with the light and dark values side by side. No other file holds a colour value.
2. Colours have meaning names (`background`, `foreground`, `primary`, `muted`, `destructive`, `border`), never appearance names. We keep shadcn's variable names so components added with the shadcn CLI work unchanged. We add our own meaning tokens (`success`, `warning`, `info`, each with a `-foreground`) in the same file.
3. Components use only the token classes (`bg-background`, `text-muted-foreground`, `border-border`). Raw colours (`bg-red-500`, `#fff`, `rgb(...)`) are not allowed. A `dark:` prefix on a colour is also not needed, because the variable already changes.
4. Typography is defined only in `typography.css` and `fonts.ts`. Fonts are loaded with `next/font` and exposed as variables. Components use the type roles (`text-display`, `text-title`, `text-heading`, `text-body`, `text-caption`), not ad hoc font sizes and weights.
5. Spacing, radius and shadows use the Tailwind scale and the radius token. No pixel values in class names (`p-[13px]`).
6. Dark mode is switched by the `dark` class from `next-themes` (`attribute="class"`, `defaultTheme="system"`, `enableSystem`). The `<html>` element has `suppressHydrationWarning`, because the provider changes its class before React hydrates.
7. Motion values (durations, easings, springs, variants) are defined only in `shared/theme/motion.ts`. Components never type a duration or an easing.
8. `MotionConfig reducedMotion="user"` wraps the app once in `app-providers.tsx`. Animations must still make sense when transforms are off.
9. Only Client Components animate, and only the wrappers in `shared/components/motion/` import `motion/react`. Feature code uses `<FadeIn>` and `<FadeUp>`, so a change of animation library touches one folder.
10. Keep animations short and rare: page and list entrances, dialogs, feedback. Nothing loops without a reason.
11. After `shadcn init` and after each `shadcn add`, check that new variables are copied into `tokens.css` and nothing is duplicated in `globals.css`.
12. Tests and review: a screen is checked in both themes before it is done.

### Colours

The palette is Ingrain's "Obsidian Mint". Hex values are used as designed. The names are meaning names, and shadcn's own names are kept so components added with the CLI work unchanged.

| Token | Light | Dark |
|---|---|---|
| `background` | #F7FAF8 | #0B1213 |
| `card`, `popover` (Surface) | #FFFFFF | #162320 |
| `muted`, `secondary`, `accent` (Subtle Surface) | #EDF5F1 | #20352E |
| `foreground` (Primary Text) | #19332C | #F0FAF5 |
| `muted-foreground` (Secondary Text) | #536D64 | #A5BBB2 |
| `border`, `input` | #DDE9E2 | #2A433A |
| `primary` (Primary Mint) | #087F65 | #27D5A0 |
| `primary-hover` | #076B55 (Deep Mint) | #49E3B4 |
| `primary-pressed` | #055541 | #1BB98C |
| `primary-disabled` | #B9CEC4 | #42665A |
| `primary-foreground` (Text on Primary) | #FFFFFF | #06251B |
| `ring` (Focus) | #087F65 | #49E3B4 |
| `success` | #16803D | #4ADE80 |
| `warning` | #A65B08 | #F6BD60 |
| `destructive` (Error) | #C63746 | #FF7584 |
| `info` | #246BC4 | #75B6FF |
| `ai` (AI Violet) | #7656C9 | #B9A0FF |

`ai` is used only for AI features, so they look the same everywhere. The text colour on a semantic fill (`success-foreground` and so on) is not in the palette. We use the Text on Primary value of each theme (white in light, #06251B in dark), and it is validated below.

```css
/* shared/theme/tokens.css: the only file that holds colour values */
:root {
  --radius: 0.625rem;

  --background: #F7FAF8;
  --foreground: #19332C;
  --card: #FFFFFF;
  --card-foreground: #19332C;
  --popover: #FFFFFF;
  --popover-foreground: #19332C;
  --muted: #EDF5F1;
  --muted-foreground: #536D64;
  --secondary: #EDF5F1;
  --secondary-foreground: #19332C;
  --accent: #EDF5F1;
  --accent-foreground: #19332C;
  --border: #DDE9E2;
  --input: #DDE9E2;

  --primary: #087F65;
  --primary-hover: #076B55;
  --primary-pressed: #055541;
  --primary-disabled: #B9CEC4;
  --primary-foreground: #FFFFFF;
  --ring: #087F65;

  --success: #16803D;
  --success-foreground: #FFFFFF;
  --warning: #A65B08;
  --warning-foreground: #FFFFFF;
  --destructive: #C63746;
  --destructive-foreground: #FFFFFF;
  --info: #246BC4;
  --info-foreground: #FFFFFF;
  --ai: #7656C9;
  --ai-foreground: #FFFFFF;
}

.dark {
  --background: #0B1213;
  --foreground: #F0FAF5;
  --card: #162320;
  --card-foreground: #F0FAF5;
  --popover: #162320;
  --popover-foreground: #F0FAF5;
  --muted: #20352E;
  --muted-foreground: #A5BBB2;
  --secondary: #20352E;
  --secondary-foreground: #F0FAF5;
  --accent: #20352E;
  --accent-foreground: #F0FAF5;
  --border: #2A433A;
  --input: #2A433A;

  --primary: #27D5A0;
  --primary-hover: #49E3B4;
  --primary-pressed: #1BB98C;
  --primary-disabled: #42665A;
  --primary-foreground: #06251B;
  --ring: #49E3B4;

  --success: #4ADE80;
  --success-foreground: #06251B;
  --warning: #F6BD60;
  --warning-foreground: #06251B;
  --destructive: #FF7584;
  --destructive-foreground: #06251B;
  --info: #75B6FF;
  --info-foreground: #06251B;
  --ai: #B9A0FF;
  --ai-foreground: #06251B;
}

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-card: var(--card);
  --color-card-foreground: var(--card-foreground);
  --color-popover: var(--popover);
  --color-popover-foreground: var(--popover-foreground);
  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);
  --color-secondary: var(--secondary);
  --color-secondary-foreground: var(--secondary-foreground);
  --color-accent: var(--accent);
  --color-accent-foreground: var(--accent-foreground);
  --color-border: var(--border);
  --color-input: var(--input);
  --color-ring: var(--ring);
  --color-primary: var(--primary);
  --color-primary-hover: var(--primary-hover);
  --color-primary-pressed: var(--primary-pressed);
  --color-primary-disabled: var(--primary-disabled);
  --color-primary-foreground: var(--primary-foreground);
  --color-success: var(--success);
  --color-success-foreground: var(--success-foreground);
  --color-warning: var(--warning);
  --color-warning-foreground: var(--warning-foreground);
  --color-destructive: var(--destructive);
  --color-destructive-foreground: var(--destructive-foreground);
  --color-info: var(--info);
  --color-info-foreground: var(--info-foreground);
  --color-ai: var(--ai);
  --color-ai-foreground: var(--ai-foreground);
  --radius-lg: var(--radius);
}
```

Interaction states are used through the token classes: `bg-primary hover:bg-primary-hover active:bg-primary-pressed`, `disabled:bg-primary-disabled`, and `focus-visible:ring-ring`.

### Contrast check (WCAG 2.x, computed from the hex values)

- Text and fills pass AA (4.5:1 or more) in both themes: primary text on background 12.9 (light) and 17.7 (dark), secondary text 5.1 to 8.0, white on Primary Mint 4.96, Text on Primary on dark mint 8.6, and the semantic text colours 5.0 to 5.3 (light) and 6.3 to 9.6 (dark).
- Primary mint as text on the background is 4.72 (light) and 10.0 (dark), so it passes AA as link or button text.
- The focus ring is 4.72 (light) and 11.65 (dark) against the background, above the 3:1 needed for focus indicators.
- Disabled fills (`primary-disabled`) are 1.66 (light) and 2.55 (dark) with their text. Disabled controls are exempt from contrast requirements, but they must not be the only way to know a control is off.
- Borders are 1.19 (light) and 1.77 (dark) against the background. This is fine for decorative dividers. A form control's outline (`input`) needs 3:1 to be identifiable, so inputs get a stronger border colour or a visible fill. This still needs a decision (open point 1 below).

### Typography

Plus Jakarta Sans is used for headings and Inter for body text and UI.

```ts
// shared/theme/fonts.ts
import { Inter, Plus_Jakarta_Sans } from "next/font/google";

export const fontHeading = Plus_Jakarta_Sans({ subsets: ["latin"], variable: "--font-plus-jakarta-sans", display: "swap" });
export const fontSans = Inter({ subsets: ["latin"], variable: "--font-inter", display: "swap" });
```

```css
/* shared/theme/typography.css */
@theme inline {
  --font-sans: var(--font-inter);
}

@utility text-display { font-family: var(--font-plus-jakarta-sans), sans-serif; font-size: 2.25rem; line-height: 2.5rem; font-weight: 700; letter-spacing: -0.02em; }
@utility text-title   { font-family: var(--font-plus-jakarta-sans), sans-serif; font-size: 1.5rem;  line-height: 2rem;    font-weight: 600; letter-spacing: -0.01em; }
@utility text-heading { font-family: var(--font-plus-jakarta-sans), sans-serif; font-size: 1.125rem; line-height: 1.75rem; font-weight: 600; }
@utility text-body    { font-family: var(--font-inter), sans-serif; font-size: 1rem;     line-height: 1.5rem;  font-weight: 400; }
@utility text-caption { font-family: var(--font-inter), sans-serif; font-size: 0.875rem; line-height: 1.25rem; font-weight: 400; }
```

The sizes and weights are starting values. Headings (`text-display`, `text-title`, `text-heading`) use Plus Jakarta Sans, and body and UI roles use Inter.

```css
/* src/app/globals.css */
@import "tailwindcss";
@import "../shared/theme/tokens.css";
@import "../shared/theme/typography.css";

@layer base {
  body { @apply bg-background text-foreground text-body; }
}
```

### Providers and layout

```tsx
// shared/theme/theme-provider.tsx
"use client";

import { ThemeProvider as NextThemesProvider } from "next-themes";

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  return (
    <NextThemesProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
      {children}
    </NextThemesProvider>
  );
}
```

```tsx
// shared/providers/app-providers.tsx
"use client";

import { MotionConfig } from "motion/react";

export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <ThemeProvider>
      <MotionConfig reducedMotion="user">{children}</MotionConfig>
    </ThemeProvider>
  );
}
```

```tsx
// src/app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning className={`${fontHeading.variable} ${fontSans.variable}`}>
      <body>
        <AppProviders>{children}</AppProviders>
      </body>
    </html>
  );
}
```

### Motion

```ts
// shared/theme/motion.ts
import type { Transition, Variants } from "motion/react";

export const MotionDuration = { FAST: 0.15, BASE: 0.25, SLOW: 0.4 } as const;

export const transitions = {
  base: { duration: MotionDuration.BASE, ease: [0.16, 1, 0.3, 1] },
  fast: { duration: MotionDuration.FAST, ease: [0.16, 1, 0.3, 1] },
  soft: { type: "spring", stiffness: 220, damping: 26 },
} satisfies Record<string, Transition>;

export const fadeVariants = {
  hidden: { opacity: 0 },
  visible: { opacity: 1, transition: transitions.base },
} satisfies Variants;

export const fadeUpVariants = {
  hidden: { opacity: 0, y: 8 },
  visible: { opacity: 1, y: 0, transition: transitions.base },
} satisfies Variants;
```

```tsx
// shared/components/motion/fade-up.tsx
"use client";

import { motion } from "motion/react";

export function FadeUp({ children, className }: { children: React.ReactNode; className?: string }) {
  return (
    <motion.div className={className} variants={fadeUpVariants} initial="hidden" animate="visible">
      {children}
    </motion.div>
  );
}
```

## 12. Shared components and patterns

Components are grouped by purpose. Variation is handled by lookup tables and variants, not by `if/else` chains, so adding a case means adding an entry and nothing existing changes.

### Folder structure

```
shared/
├── components/
│   ├── ui/          shadcn primitives made by the CLI (button, input, dialog, badge)
│   ├── buttons/     our buttons built on ui/ (submit-button, icon-button, theme-toggle)
│   ├── forms/       form-field (label + input + errors), field-errors
│   ├── feedback/    error-screen, not-found-screen, empty-state, badge (with our variants), spinner
│   ├── layout/      page-header, container, section
│   └── motion/      fade-in, fade-up (the only files that import motion/react)
└── utils/
    └── cn.ts        class-name helper used by shadcn components
```

### Rules

1. `ui/` holds the shadcn primitives. We change them only to adjust the base look or add a variant. They hold no business logic and know nothing about features.
2. Our own components are built from `ui/` primitives and placed by purpose (`buttons/`, `forms/`, `feedback/`, `layout/`, `motion/`). A new group is added only when three or more components share a purpose.
3. Dependencies point one way: feature components use `shared/components`, and inside it `buttons`, `forms`, `feedback` and `layout` use `ui`. `ui` uses nothing from the other groups.
4. `shared/components` never imports from `features`. Anything that knows about orders, payments or users lives in that feature's `components/`.
5. Variation is a variant, not a branch. Variants come from `cva` (shadcn already uses it). To add a look, add a variant entry. Do not add another boolean prop and another `if`.
6. When a value decides what is shown (a status, a type, a role), the component uses a lookup table `Record<Value, Entry>`, declared `as const satisfies Record<Value, Entry>`. A new value without an entry fails the build. A `switch` or an `if/else` chain over a value is not used for this.
7. A component has at most one boolean prop that changes how it looks. More than that means separate components or a variant.
8. Prefer composition: a component takes `children` or named slots for the parts that vary, instead of a prop for every option.
9. A component is a Server Component unless it needs the browser (section 7). Animation wrappers and the theme toggle are Client Components.
10. Component and file names follow section 1 (kebab-case file, `PascalCase` export). One exported component per file.
11. shadcn is configured in `components.json` so its generated files go to `shared/components/ui` and `cn` to `shared/utils/cn.ts`. Run the CLI from the project root.
12. Components that move in this section: `ErrorScreen` and `NotFoundScreen` are in `feedback/`, and `FieldErrors` is in `forms/`.

### Lookup table instead of if/else

```ts
// features/orders/constants/order-status.ts (see section 10)
export const OrderStatus = { PENDING: "PENDING", PAID: "PAID", SHIPPED: "SHIPPED", CANCELLED: "CANCELLED" } as const;
export type OrderStatus = (typeof OrderStatus)[keyof typeof OrderStatus];
```

```tsx
// features/orders/components/order-status-badge.tsx
import { Badge } from "@/shared/components/feedback/badge";

type StatusEntry = { labelKey: OrderMessageKey; variant: BadgeVariant };

const STATUS_ENTRIES = {
  PENDING:   { labelKey: "order.status.pending",   variant: "warning" },
  PAID:      { labelKey: "order.status.paid",      variant: "success" },
  SHIPPED:   { labelKey: "order.status.shipped",   variant: "info" },
  CANCELLED: { labelKey: "order.status.cancelled", variant: "destructive" },
} as const satisfies Record<OrderStatus, StatusEntry>;

export function OrderStatusBadge({ status }: { status: OrderStatus }) {
  const { labelKey, variant } = STATUS_ENTRIES[status];
  return <Badge variant={variant}>{tOrders(labelKey)}</Badge>;
}
```

The status labels (`order.status.pending`, and so on) are keys in `orders.messages.ts`. Adding a status `REFUNDED` means adding it to `OrderStatus`. The build then fails on `STATUS_ENTRIES` until its entry is added, and `OrderStatusBadge` never changes.

### Variants with cva

```tsx
// shared/components/feedback/badge.tsx
import { cva, type VariantProps } from "class-variance-authority";

const badgeVariants = cva("inline-flex items-center rounded-md px-2 py-0.5 text-caption", {
  variants: {
    variant: {
      default: "bg-primary text-primary-foreground",
      success: "bg-success text-success-foreground",
      warning: "bg-warning text-warning-foreground",
      info: "bg-info text-info-foreground",
      destructive: "bg-destructive text-destructive-foreground",
    },
  },
  defaultVariants: { variant: "default" },
});

export type BadgeVariant = NonNullable<VariantProps<typeof badgeVariants>["variant"]>;

export function Badge({ variant, className, ...props }: React.ComponentProps<"span"> & { variant?: BadgeVariant }) {
  return <span className={cn(badgeVariants({ variant }), className)} {...props} />;
}
```

### A button built on the primitive

```tsx
// shared/components/buttons/submit-button.tsx
import { Button } from "@/shared/components/ui/button";

export function SubmitButton({ pending, children }: { pending: boolean; children: React.ReactNode }) {
  return (
    <Button type="submit" disabled={pending}>
      {children}
    </Button>
  );
}
```

The form passes `pending` from `useActionState` (section 5), and the label is a message key resolved by the caller.

## 13. Component size, composition and function rules

Two goals. A component file reads like a short list of named parts. A function does one thing, and you can understand it without scrolling.

### Components: how a big one is split

A component stays one file until it has parts. Then it becomes a folder named after it. The main file has the same name as the folder and reads like a list of its parts.

```
features/orders/components/
├── order-status-badge.tsx          small, stays one file
├── order-details/
│   ├── order-details.tsx           reads like a list of the parts below
│   ├── order-header.tsx
│   ├── order-amount.tsx
│   └── order-actions.tsx
└── order-list/
    ├── order-list.tsx
    ├── order-list-row.tsx
    └── order-list-pagination.tsx
```

```tsx
// features/orders/components/order-details/order-details.tsx
export function OrderDetails({ order }: { order: Order }) {
  return (
    <article>
      <OrderHeader id={order.id} status={order.status} createdAt={order.createdAt} />
      <OrderAmount amount={order.amount} currency={order.currency} />
      <OrderActions orderId={order.id} status={order.status} />
    </article>
  );
}
```

Reading it gives the story of the screen: header, amount, actions. Each part's own markup and details are in its own small file.

### Component rules

1. A component has one visible job and one reason to change.
2. Size guide: about 80 lines per component file, markup included, and about 7 direct children. Past that, split it. A file may be longer only when splitting would hide the story, and the reviewer agrees.
3. Split by meaning, not by line count. A part becomes its own component when it has a name you would say aloud ("order header"), has its own state or logic, or repeats. Do not cut a file into pieces that have no name.
4. The parent reads like a list. Its body is named components, with little logic and almost no raw markup. Layout wrappers (`<article>`, `<section>`) are fine.
5. A part receives only what it shows. Pass the fields it uses, or the model when it shows most of it. Do not pass data through components that do not use it. Use `children` for that.
6. One file while it is small. A folder only when it has parts. No folder with a single file, and no `index.ts` inside a component folder. Import the main file by its path.
7. A part used by one component stays in that component's folder. When a second component needs it, it moves up to the feature's `components/`. When a second feature needs it, it moves to `shared/components/`. Move it when the need is real, not before.
8. At most two folder levels under `components/`. If it needs more, the component is too big and should be rethought.
9. Components render, they do not compute. Event handlers and state of a Client Component that grow past a few lines move to a function or a `use-<name>.ts` file next to the component.
10. Differences between cases use a lookup table or a variant (section 12), not an `if/else` chain in the markup.
11. Names: the folder is kebab-case, the component is `PascalCase`, and the parts start with the parent's noun (`order-header`, `order-amount`), so a search finds them together.
12. Do not over-split. If the parent would become a list of one-line wrappers, keep it in one file. Three small related lines do not need a file.
13. The same rules apply to `page.tsx` and to components in `shared/components`.

### Function rules (logic that is not presentation)

These apply to api functions, mappers, actions, rules, helpers and hooks.

1. One function, one responsibility. The name says it. If the name needs "and", it is two functions.
2. Return early. Check the failing cases first with guard clauses, then write the normal path without extra nesting. No `else` after a `return`. Nesting is at most two levels deep.
3. Size guide: about 20 lines. A longer function is split into named steps. A function that coordinates (an action, an orchestrating api function) reads like a list of steps, and each step is a named function.
4. Meaningful names. A function is a verb and a noun (`parseSearchParams`, `fieldErrorsFromViolations`). A boolean starts with `is`, `has`, `can` or `should`. No `data`, `info`, `item`, `handle`, `process` or `doStuff`. No unexplained abbreviations. `id` and `dto` are fine.
5. Consistent words. One word per operation across the code: `get` for one item, `search` for a list, `create`, `update`, `cancel`, `format`, `parse`, `to` for conversions (`<Source>To<Target>`). We do not mix `fetch`, `load` and `get` for the same thing.
6. At most three parameters. More than that takes one object. A boolean parameter that switches the behaviour is not allowed. Write two functions.
7. Side effects stay at the edges. Mappers, rules, formatters and helpers are pure: the same input gives the same output, and they do not change their input. Network calls, redirects and cookies happen in api functions and actions only.
8. A function either returns a value or does something. It does not do both quietly. A function that can fail in an expected way returns the result (`FormState`, `ActionResult`). Unexpected failures throw (section 4).
9. Comments say why, not what. If a comment explains what the code does, rename or split the code instead.
10. No magic values (section 10) and no hardcoded text (section 8).

### Example: the same action, split by these rules

The create-order action (section 5) coordinates five steps. The step that turns a backend error into a form state is its own function, so the action reads as a list and has no nested `if` blocks.

```ts
// features/orders/actions/create-order.action.ts
"use server";

export async function createOrderAction(_prev: FormState, formData: FormData): Promise<FormState> {
  const values = formValuesFrom(formData);

  const parsed = createOrderSchema.safeParse(values);
  if (!parsed.success) {
    return {
      status: "error",
      fieldErrors: translateFieldErrors(z.flattenError(parsed.error).fieldErrors, orderMessages),
      values,
    };
  }

  const violations = runRules(createOrderRules, parsed.data);
  if (violations.length > 0) {
    return { status: "error", fieldErrors: fieldErrorsFromViolations(violations, orderMessages), values };
  }

  let order: Order;
  try {
    order = await createOrder(createOrderFormToCreateOrderRequestDto(parsed.data));
  } catch (e) {
    return backendErrorToFormState(e, values);
  }

  redirect(ROUTES.ORDERS.DETAIL(order.id)); // outside the try/catch
}

function backendErrorToFormState(e: unknown, values: Record<string, string>): FormState {
  if (!(e instanceof ApiError)) throw e; // goes to error.tsx

  if (e.status === HttpStatus.BAD_REQUEST) {
    return { status: "error", fieldErrors: fieldErrorsFrom(e.details), values };
  }
  if (e.status === HttpStatus.CONFLICT || e.status === HttpStatus.UNPROCESSABLE) {
    return { status: "error", formError: e.message, values };
  }
  throw e;
}
```

### Open points for the palette

1. Input outlines: `input` is the same colour as `border` (1.19:1 in light). Choose a stronger input border or a filled input before building forms.
2. The semantic text colours (`success-foreground` and the others) and the type sizes and weights are assumptions, not part of the supplied palette.
3. The palette notes say the interaction and disabled tokens still need final component-level validation. The checks above cover text and focus contrast only.
