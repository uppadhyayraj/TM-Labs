# Act 3 — The Governed Framework
### Democratizing API Testing with AI

> **Goal:** Make AI generation deterministic, self-healing, and observable —  
> by giving the AI an architecture it can read and operate within.

---

## What you discovered in Acts 1 and 2

**Act 1:** AI without structure generates something different every time.  
Green report. Passing tests. Invisible problem.

**Act 2:** One base class makes AI generation consistent.  
Same pattern. Same shape. Every person, every run.

**Act 3:** Consistency is not enough on its own.

Consider what still breaks in Act 2:
- The AI can still invent test data instead of using shared constants
- The AI can still manage auth inline instead of using a fixture
- If the API schema changes, assertions fail and there is no recovery mechanism
- A new team member gets consistent *structure* — but not consistent *behaviour*

> The difference between Act 2 and Act 3 is the difference between  
> *a developer following a convention* and *a framework enforcing one.*

---

## What you are doing in this act

You will add three more files to the project — test data, a fixture, and response types.  
Then you will run `/init` and let Copilot read the project and write its own operating instructions.  
From that point, every generation is governed — not by convention, but by contract.

---

## Step 1 — Add test data as constants

Create this file **yourself**:

```
src/
└── data/
    └── testData.ts
```

```typescript
// src/data/testData.ts

/**
 * Test data as first-class constants.
 * The AI will use these instead of generating inline data.
 */
export const TEST_USER = {
  name:     'Alex Demo',
  email:    'testuser@mybasket.com',
  password: 'P@ssw0rd!',
  phone:    '021-555-0101',
};

export const DELIVERY_ADDRESS = {
  street:   '42 Queen Street',
  city:     'Auckland',
  postcode: '1010',
  country:  'NZ',
};

export const PAYMENT = {
  type:        'credit_card',
  card_number: '4111 1111 1111 1111',
  expiry:      '12/28',
  cvv:         '123',
};

export const DUPLICATE_USER = {
  email:    'duplicate@mybasket.com',
  password: 'P@ssw0rd!',
};
```

**Why this matters:**  
In Act 1 and Act 2, the AI invented test data each time it generated tests.  
Different names, different emails, sometimes different formats.  
This file makes test data a project decision — not an AI decision.

---

## Step 2 — Add the auth fixture

Create this file **yourself**:

```
src/
└── fixtures/
    └── api.fixtures.ts
```

```typescript
// src/fixtures/api.fixtures.ts
import { test as base }  from '@playwright/test';
import { UserAPI }       from '../apis/UserAPI';
import { TEST_USER }     from '../data/testData';

/**
 * Shared auth fixture.
 * Login once per test suite — reuse the token across all tests.
 * Tests that need auth declare `authToken` as a parameter.
 */
type AuthFixtures = { authToken: string };

export const test = base.extend<AuthFixtures>({
  authToken: async ({ request }, use) => {
    const userAPI   = new UserAPI(request);
    const response  = await userAPI.login(TEST_USER.email, TEST_USER.password);
    const { token } = await response.json();
    await use(token);
  },
});

export { expect } from '@playwright/test';
```

**Why this matters:**  
In Act 1, the JWT token was handled differently in every file — sometimes hardcoded,  
sometimes extracted inline, sometimes missing entirely.  
This fixture makes auth a single shared responsibility.  
Any test that needs a token declares `{ authToken }` and gets it — without knowing how.

---

## Step 3 — Add response types

Create this file **yourself**:

```
src/
└── types/
    └── api.types.ts
```

```typescript
// src/types/api.types.ts

/**
 * Response interfaces for all MyBasket API endpoints.
 * Shared across page objects and test specs.
 */
// ---------------------------------------------------------------------------
// Shared / utility
// ---------------------------------------------------------------------------

export interface ApiError {
  error:    string;
  details?: object[];
}

// ---------------------------------------------------------------------------
// Gateway
// ---------------------------------------------------------------------------

export interface ServiceHealth {
  status:       string;
  responseTime: number;
}

export interface HealthResponse {
  gateway:  string;
  status:   'healthy' | 'unhealthy';
  services: {
    'product-service': ServiceHealth;
    'cart-service':    ServiceHealth;
    'order-service':   ServiceHealth;
    'ai-service':      ServiceHealth;
    'user-service':    ServiceHealth;
  };
  timestamp: string;
}

export interface GatewayService {
  name: string;
  path: string;
}

export interface GatewayInfo {
  gateway:   string;
  version:   string;
  services:  GatewayService[];
  timestamp: string;
}

// ---------------------------------------------------------------------------
// Users / Auth
// ---------------------------------------------------------------------------

export interface UserPublic {
  id:        string;
  username:  string;
  name:      string;
  email:     string;
  createdAt: string;
  updatedAt: string;
}

export interface AuthResponse {
  user:  UserPublic;
  token: string;
}

export interface RegisterRequest {
  username: string;
  password: string;
  name:     string;
  email:    string;
}

export interface LoginRequest {
  username: string;
  password: string;
}

export interface UpdateUserRequest {
  name?:     string;
  email?:    string;
  password?: string;
}

// ---------------------------------------------------------------------------
// Products
// ---------------------------------------------------------------------------

export interface Product {
  id:          string;
  name:        string;
  price:       number;
  description: string;
  image:       string;
  dataAiHint:  string;
  category?:   string;
  inStock:     boolean;
}

export interface Pagination {
  total:      number;
  page:       number;
  limit:      number;
  totalPages: number;
}

export interface ProductList {
  products:   Product[];
  pagination: Pagination;
}

// ---------------------------------------------------------------------------
// Cart
// ---------------------------------------------------------------------------

export interface CartItem {
  id:           string;
  name:         string;
  price:        number;
  description:  string;
  image:        string;
  dataAiHint:   string;
  quantity:     number;
  addedAt:      string;
}

export interface Cart {
  id:          string;
  userId:      string;
  items:       CartItem[];
  totalAmount: number;
  totalItems:  number;
  createdAt:   string;
  updatedAt:   string;
}

export interface AddToCartRequest {
  productId:  string;
  quantity?:  number;
}

export interface UpdateCartItemRequest {
  quantity: number;
}

// ---------------------------------------------------------------------------
// Orders
// ---------------------------------------------------------------------------

export interface Address {
  street:  string;
  city:    string;
  state:   string;
  zipCode: string;
  country: string;
}

export interface PaymentMethod {
  type:   'credit_card' | 'debit_card' | 'paypal' | 'apple_pay' | 'google_pay';
  last4?: string;
  brand?: string;
}

export interface OrderItem {
  id:           string;
  name:         string;
  price:        number;
  quantity:     number;
  image?:       string;
  description?: string;
  dataAiHint?:  string;
}

export type OrderStatus =
  | 'pending'
  | 'confirmed'
  | 'processing'
  | 'shipped'
  | 'delivered'
  | 'cancelled';

export interface Order {
  id:                 string;
  userId:             string;
  items:              OrderItem[];
  totalAmount:        number;
  status:             OrderStatus;
  shippingAddress:    Address;
  billingAddress:     Address;
  paymentMethod:      PaymentMethod;
  trackingNumber?:    string;
  estimatedDelivery?: string;
  actualDelivery?:    string;
  createdAt:          string;
  updatedAt:          string;
}

export interface OrderList {
  orders:     Order[];
  pagination: Pagination;
}

export interface CreateOrderRequest {
  items:           OrderItem[];
  shippingAddress: Address;
  billingAddress:  Address;
  paymentMethod:   PaymentMethod;
}

// ---------------------------------------------------------------------------
// Recommendations
// ---------------------------------------------------------------------------

export interface Suggestion {
  name:       string;
  reason:     string;
  category:   string;
  confidence: number;
}

export interface GrocerySuggestionsRequest {
  cartItems?: string[];
}

export interface GrocerySuggestionsResponse {
  suggestions: Suggestion[];
  generatedAt: string;
  confidence:  number;
}

export interface PersonalizedRecommendationsRequest {
  cartItems?:      string[];
  userId?:         string;
  maxSuggestions?: number;
}

export interface PersonalizedRecommendationsResponse {
  suggestions: Suggestion[];
  userId:      string;
  generatedAt: string;
  confidence:  number;
}
```

**Why this matters:**  
Without types, the AI guesses response shapes. Guesses drift over time.  
With types, every page object and every test spec works with the same  
definition of what an API response looks like.

---

## Step 4 — Verify your project structure

Before running `/init`, your project should look exactly like this:

```
your-project/
├── .vscode/
│   └── mcp.json
├── playwright.config.ts
├── test-plan.md
├── src/
│   ├── apis/
│   │   ├── BaseAPI.ts          ← from Act 2
│   │   ├── UserAPI.ts          ← from Act 2
│   │   ├── ProductAPI.ts       ← from Act 2
│   │   ├── CartAPI.ts          ← from Act 2
│   │   └── OrderAPI.ts         ← from Act 2
│   ├── data/
│   │   └── testData.ts         ← you just created this
│   ├── fixtures/
│   │   └── api.fixtures.ts     ← you just created this
│   └── types/
│       └── api.types.ts        ← you just created this
└── tests/
    └── api/
        └── [act 2 generated specs]
```

If your structure matches this, continue to Step 5.

---

## Step 5 — Run `/init`

Type this in Copilot Chat. Nothing else — no additional prompt:

```
/init
```

Wait for Copilot to finish scanning the project.

**It will generate a file at `.github/copilot-instructions.md`**

Open that file and read it carefully.

**What to look for:**

- Did it detect `BaseAPI.ts`? Does it say all page objects must extend it?
- Did it detect `testData.ts`? Does it say to use exported constants instead of inline data?
- Did it detect `api.fixtures.ts`? Does it say to use the `authToken` fixture?
- Did it detect `tests/api/`? Does it say to place all specs there?
- Did it detect `playwright.config.ts`? Does it say to use the configured `baseURL`?

> **You did not write a single line of that file.**  
> Copilot read your project architecture and wrote its own operating instructions.  
>
> The quality of that file depends entirely on the quality of your project.  
> A well-structured project produces precise instructions.  
> A poorly structured project produces vague ones.  
> This is why the files in Steps 1–3 matter.

---

## Step 6 — Governed generation

```
/test-generation Generate the full Playwright TypeScript test suite
for MyBasket. Use the page objects in src/apis/, the auth fixture
from src/fixtures/api.fixtures.ts, and test data from
src/data/testData.ts. Place all specs in tests/api/
```

**Check the generated files against these questions:**

1. Does it import from `testData.ts` — or does it have inline data like `email: 'user@test.com'`?
2. Does it import `test` from `api.fixtures.ts` — or does it manage auth inline?
3. Are response types imported from `api.types.ts`?
4. Are all files in `tests/api/`?

**Compare your answers with your neighbour.**

> This time the answer should be the same for everyone.

---

## Step 7 — Break a test, then heal it

The presenter will now simulate a schema change in the MyBasket Order service.

One field in the order response will be renamed: `order_id` → `orderId`

Run your tests:

```bash
npx playwright test tests/api/
```

**One assertion will fail.** Note which test and which line.

Now run:

```
/test-healing The order placement assertion is failing after a schema
change. Detect the mismatch and patch the assertion to match the
current API response. Re-run to confirm green.
```

**What to look for:**
- The AI finds the failing assertion without being told which file
- It patches only that assertion — nothing else changes
- Re-running confirms all tests pass

> The healing skill can do this because the governed framework  
> gave it enough context to know what "correct" looks like.  
> In Act 1, there was no contract to heal back to.

---

## Step 8 — Final HTML report

```
/test-execution Generate the HTML report for the governed run.
Save to ./output/governed-report.html
```

Open both reports side by side:
- `output/act1-report.html`
- `output/governed-report.html`

**Same coverage. Same requests. Same responses.**  
Different architecture underneath.

---

## 🔴 Final comparison — three acts

Fill this in for yourself:

| Question | Act 1 | Act 2 | Act 3 |
|---|---|---|---|
| Does re-running generate the same output? | | | |
| Does a new team member get the same output? | | | |
| Is test data consistent across all tests? | | | |
| Is auth handled in one place? | | | |
| Can the AI recover from a schema change? | | | |
| Is the output observable and shareable? | | | |

---

## What you have at the end of Act 3

```
your-project/
├── .vscode/mcp.json
├── .github/
│   └── copilot-instructions.md   ← generated by /init
├── playwright.config.ts
├── test-plan.md
├── src/
│   ├── apis/
│   │   ├── BaseAPI.ts
│   │   ├── UserAPI.ts
│   │   ├── ProductAPI.ts
│   │   ├── CartAPI.ts
│   │   └── OrderAPI.ts
│   ├── data/
│   │   └── testData.ts
│   ├── fixtures/
│   │   └── api.fixtures.ts
│   └── types/
│       └── api.types.ts
├── tests/
│   └── api/
│       └── [governed generated specs]
└── output/
    ├── act1-report.html
    └── governed-report.html
```

---

## The answer to the question we started with

> *"AI can generate tests in minutes. But will you trust them in three months?"*

| | Answer |
|---|---|
| Act 1 — no structure | No. Different output every run. Invisible inconsistency. |
| Act 2 — base class | Maybe. Consistent structure. But data and auth still drift. |
| Act 3 — governed framework | Yes. Deterministic generation. Self-healing. Observable. |

> **The AI did not get smarter between Act 1 and Act 3.**  
> **The engineering got better.**  
>
> That is the lesson. The MCP server is the vehicle.  
> The engineering principles are the point.

---

## What you can use from tomorrow

| What | How to start |
|---|---|
| MCP server with any project | Add `@democratize-quality/mcp-server` to `.vscode/mcp.json` |
| Test planning from any API | Point `/api-planning` at any `swagger.json` or `/api-docs.json` |
| Governed generation | Add `BaseAPI.ts`, run `/init`, then `/test-generation` |
| Live E2E execution | Use `/test-execution` with a session ID — JWT chaining is automatic |
| Test healing | Use `/test-healing` when schema changes break assertions |
| HTML reports | Every `/test-execution` run produces a shareable report |

---

## Further reading

*Scalable Test Automation with Playwright* — Raj Uppadhyay  
bpbonline.com/products/scalable-test-automation-with-playwright

| Chapter | What it covers | Where you saw it today |
|---|---|---|
| Ch.1 — Design Patterns | Base class patterns for API automation | `BaseAPI.ts` → consistent AI generation |
| Ch.4 — Framework Migration | Keeping tests alive through API change | `/test-healing` → schema drift recovery |
| Ch.5 — Engineering Principles | Separation of concerns, single responsibility | POM structure before the AI ran |
| Ch.7 — Test Data & Config | Test data and fixtures as first-class citizens | `testData.ts`, `api.fixtures.ts`, `/init` |
| Ch.10 — Observability | Governance evidence, not just test results | HTML report → full request/response audit |

---

*Workshop by Raj Uppadhyay — `@democratize-quality/mcp-server`*
