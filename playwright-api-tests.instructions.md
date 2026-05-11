---
description: Use when creating Playwright API tests, adding new API page objects, adding test methods, or generating test specs for My Basket App endpoints. Covers project structure, page object pattern, auth flow, naming conventions, and spec file conventions.
applyTo: "tests/**/*.spec.ts", "src/apis/**/*.ts"
---

# My Basket App — Playwright API Test Conventions

## Must-Have Rules (Non-Negotiable)

These rules apply to every spec file and every page object. Violations must be corrected before a test is considered complete.

### 1 — Base URL comes from `playwright.config.ts` only
`playwright.config.ts` sets `use.baseURL: 'http://localhost:3000'`. `BaseAPI` picks this up as its default.

- **Never** hardcode `http://localhost:3000` (or any origin) in a spec file.
- **Never** pass a second argument to a page-object constructor in tests — let `BaseAPI` use its default.
- To change the target environment, edit `playwright.config.ts`; no spec file should need touching.

```ts
// ✅ correct — no base URL in the spec
const api = new ProductAPI(request);

// ❌ wrong — hardcoded origin
const api = new ProductAPI(request, 'http://localhost:3000');
const res = await request.get('http://localhost:3000/api/products');  // also wrong
```

### 2 — All test data comes from `src/data/TestData.ts`
Import constants from `src/data/TestData.ts` for every piece of fixed test data (credentials, addresses, payment info). Do not write inline literals for data that already exists in that file.

```ts
// ✅ correct
import { TEST_USER, DELIVERY_ADDRESS, PAYMENT } from '../src/data/TestData';

// ❌ wrong — inline data that duplicates TestData.ts
const payload = { email: 'testuser@mybasket.com', password: 'P@ssw0rd!' };
```

For data that must be unique per run (usernames, emails), derive it from the exported constants:
```ts
const suffix   = Date.now();
const username = `user_${suffix}`;
const email    = `user_${suffix}@mybasket.com`;
// passwords, addresses, payment → always from TestData.ts
```

### 3 — All API calls go through page objects (`src/apis/`)
Spec files must **never** call `request.get/post/put/delete` directly. Every HTTP interaction must be routed through the appropriate page-object class from `src/apis/`.

```ts
// ✅ correct — via page object
const api = new CartAPI(request);
const res = await api.addItem(userId, token, { productId, quantity: 1 });

// ❌ wrong — raw request in spec
const res = await request.post(`/api/cart/${userId}/items`, { data: { productId } });
```

### 4 — Response bodies must be typed using `src/types/api.types.ts`
Cast every `await res.json()` call to the matching interface from `src/types/api.types.ts`. Make assertions against typed properties, not bare `any`.

```ts
// ✅ correct
import type { AuthResponse, Product, Cart } from '../src/types/api.types';

const body = await loginRes.json() as AuthResponse;
expect(body.user.id).toBeTruthy();
expect(body.token).toBeTruthy();

const products = (await productsRes.json()) as { products: Product[]; total: number; page: number; limit: number; totalPages: number };
expect(products.products[0].inStock).toBe(true);

// ❌ wrong — untyped any
const body = await res.json();
expect(body.user.id).toBeTruthy();  // no type safety
```

---

## Project Structure

```
src/apis/          ← Page objects (one class per resource, extends BaseAPI)
src/data/          ← Test data constants (TestData.ts — single source of truth)
src/types/         ← Shared TypeScript interfaces (api.types.ts)
src/fixtures/      ← Custom Playwright fixtures (api.fixtures.ts)
tests/             ← Spec files (one per endpoint + E2E suites)
playwright.config.ts  ← baseURL and global config — the only place for base URL
```

## Adding a New Page Object (`src/apis/`)

1. Create `src/apis/<Resource>API.ts`, extend `BaseAPI`.
2. Declare local payload interfaces at the top of the file.
3. Each method returns `Promise<APIResponse>` — **never call `.json()` inside the page object**.
4. Inject `token` as a parameter on protected methods — never store it in class state.
5. Use `this.url('/path')` — never build absolute URLs manually.

```ts
import { APIResponse } from '@playwright/test';
import { BaseAPI } from './BaseAPI';

export interface CreateFooPayload { name: string; }

export class FooAPI extends BaseAPI {
  /** POST /api/foos — Create a foo. Returns 201. */
  async createFoo(token: string, payload: CreateFooPayload): Promise<APIResponse> {
    return this.request.post(this.url('/api/foos'), {
      headers: this.authHeader(token),
      data: payload,
    });
  }

  /** GET /api/foos/{id} — Get a foo. Returns 200. */
  async getFoo(id: string, token: string): Promise<APIResponse> {
    return this.request.get(this.url(`/api/foos/${id}`), {
      headers: this.authHeader(token),
    });
  }
}
```

### `BaseAPI` helpers
- `this.url('/path')` → resolves against `baseURL` from `playwright.config.ts`
- `this.authHeader(token)` → `{ Authorization: 'Bearer <token>' }`

## Adding Types (`src/types/api.types.ts`)

- This file is the **single source of truth** for all response shapes.
- It is generated from `http://localhost:3000/api-docs.json`. Fetch the schema before adding or changing types.
- Always import as a `type` import in specs: `import type { ... } from '../src/types/api.types'`.
- Use it for type assertions in tests via `as MyType` or explicit variable types, **not** inside page objects.

## Writing a Spec File (`tests/`)

### Naming
- Single-endpoint specs: `<n>-<method>-<route-slug>.spec.ts` (e.g., `13-post-apicartuseriditems.spec.ts`)
- E2E / multi-step suites: descriptive name (e.g., `happy-path-e2e.spec.ts`)

### Single-Endpoint Spec Template

```ts
// GET /api/products — Get all products
import { test, expect } from '@playwright/test';
import { ProductAPI } from '../src/apis/ProductAPI';
import type { Product } from '../src/types/api.types';

test.describe('GET /api/products', () => {
  test('returns 200 + product list', async ({ request }) => {
    const api = new ProductAPI(request);           // no base URL — from playwright.config.ts
    const res  = await api.getProducts({ page: 1, limit: 10 });
    expect(res.status()).toBe(200);
    const body = await res.json() as { products: Product[]; total: number; page: number; limit: number; totalPages: number };
    expect(body.products).toBeInstanceOf(Array);
    expect(body.products[0]).toHaveProperty('id');
    expect(body.products[0]).toHaveProperty('inStock');
  });
});
```

### Auth Flow in Specs
Use `TEST_USER` from `src/data/TestData.ts` for the password and base profile data. Always suffix the username/email with `Date.now()` to guarantee uniqueness.

```ts
import { UserAPI } from '../src/apis/UserAPI';
import type { AuthResponse } from '../src/types/api.types';
import { TEST_USER } from '../src/data/TestData';

const userApi  = new UserAPI(request);
const suffix   = Date.now();
const username = `user_${suffix}`;
const email    = `user_${suffix}@mybasket.com`;

await userApi.register({
  username,
  password: TEST_USER.password,
  name:     TEST_USER.name,
  email,
});

const loginRes          = await userApi.login({ username, password: TEST_USER.password });
const { token, user }   = await loginRes.json() as AuthResponse;
const userId            = user.id;
```

### Multi-Step / E2E Specs
Use `test.describe.configure({ mode: 'serial' })` **and** `test.beforeAll` to share auth state. Without serial mode, `--last-failed` re-runs skip setup steps, breaking downstream tests.

```ts
import { test, expect } from '@playwright/test';
import { UserAPI } from '../src/apis/UserAPI';
import type { AuthResponse } from '../src/types/api.types';
import { TEST_USER, DELIVERY_ADDRESS, PAYMENT } from '../src/data/TestData';

test.describe('My Journey', () => {
  test.describe.configure({ mode: 'serial' });

  let token:  string;
  let userId: string;

  test.beforeAll(async ({ request }) => {
    const userApi  = new UserAPI(request);
    const suffix   = Date.now();
    const username = `u_${suffix}`;
    await userApi.register({
      username,
      password: TEST_USER.password,
      name:     TEST_USER.name,
      email:    `${username}@mybasket.com`,
    });
    const res         = await userApi.login({ username, password: TEST_USER.password });
    const body        = await res.json() as AuthResponse;
    token  = body.token;
    userId = body.user.id;
  });

  test('Step 1: ...', async ({ request }) => { /* uses token, userId, DELIVERY_ADDRESS, PAYMENT */ });
  test('Step 2: ...', async ({ request }) => { /* uses token, userId */ });
});
```

## Known Quirks

### Flat Pagination on `GET /api/products`
The response is `{ products: Product[], total, page, limit, totalPages }` — pagination fields are **at the top level**, not nested under `pagination`:
```ts
// ✅
expect(body.page).toBe(1);
expect(body).toHaveProperty('totalPages');

// ❌ wrong — no nested pagination object
expect(body.pagination.page).toBe(1);
```
`GET /api/orders/{userId}` does use a nested `pagination` object (per `OrderList` schema).

### Username Uniqueness
Always suffix usernames with `Date.now()`. Registering a duplicate username returns 400; in `beforeAll` this is safe to ignore. Re-running without serial mode creates a new suffix so the user won't exist → login fails → `.user.id` throws.

### Public vs Protected Endpoints
No auth needed: `GET /health`, `GET /info`, `GET /api/products`, `GET /api/products/{id}`, `POST /api/users/register`, `POST /api/users/login`.  
All other endpoints require `Authorization: Bearer <token>`.

## Run Commands

```bash
npx playwright test                                    # all tests
npx playwright test tests/happy-path-e2e.spec.ts      # single file
npx playwright test --last-failed                      # re-run failures
npx playwright show-report api-test-reports/playwright-report
```

> The local server must be running at `http://localhost:3000` before executing tests.

## Reference Files
- Page object pattern: [src/apis/CartAPI.ts](../../src/apis/CartAPI.ts), [src/apis/OrderAPI.ts](../../src/apis/OrderAPI.ts)
- E2E serial-mode pattern: [tests/happy-path-e2e.spec.ts](../../tests/happy-path-e2e.spec.ts)
- All API types: [src/types/api.types.ts](../../src/types/api.types.ts)
- Test data constants: [src/data/TestData.ts](../../src/data/TestData.ts)
- Base URL config: [playwright.config.ts](../../playwright.config.ts)
- OpenAPI schema: `http://localhost:3000/api-docs.json`
