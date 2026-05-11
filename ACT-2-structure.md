# Act 2 — The Structure Problem
### Democratizing API Testing with AI

> **Goal:** Understand why Act 1 breaks at scale — and see how one file  
> transforms AI generation from unpredictable to consistent.

---

## What you discovered in Act 1

When the room compared outputs, everyone had run the same prompt against the same API —  
and gotten something different.

- Different file names
- Different ways of handling auth
- Different approaches to the base URL
- Some files shared code, some didn't

Every output worked. Every report was green. The problem was invisible  
until you compared notes.

> **This is the most dangerous kind of technical debt.**  
> It doesn't fail. It just quietly makes your test suite unmaintainable.

---

## What you are doing in this act

You will add a single file — `BaseAPI.ts` — to your project.  
Then you will ask the AI to generate the same tests again.

Same MCP server. Same skills. Same API.  
Completely different outcome.

---

## Before you start

Your project should still have everything from Act 1.  
Do not delete anything — you will be able to compare before and after.

---

## Step 1 — Create the foundation file

Create this folder and file **yourself**. Do not ask the AI to create it.

```
your-project/
└── src/
    └── apis/
        └── BaseAPI.ts     ← create this now
```

Copy this exactly:

```typescript
// src/apis/BaseAPI.ts
import { APIRequestContext } from '@playwright/test';

/**
 * BaseAPI — wraps Playwright's APIRequestContext.
 * Provides shared baseUrl and bearer-token helper
 * used by all resource API objects.
 */
export class BaseAPI {
  protected readonly baseUrl: string;
  protected readonly request: APIRequestContext;

  constructor(
    request: APIRequestContext,
    baseUrl: string = 'http://localhost:3000'
  ) {
    this.request = request;
    this.baseUrl = baseUrl;
  }

  /** Returns Authorization header when a token is provided. */
  protected authHeader(token: string): Record<string, string> {
    return { Authorization: `Bearer ${token}` };
  }

  /** Builds an absolute URL from a path segment. */
  protected url(path: string): string {
    return `${this.baseUrl}${path}`;
  }
}
```

**Three things this file provides:**
1. A shared `request` context — every API class uses the same Playwright request instance
2. `authHeader(token)` — one place to manage Bearer token headers
3. `url(path)` — one place to manage the base URL

> That is 39 lines. No magic. No framework.  
> Just three things every API page object will need — in one place.

---

## Step 2 — Ask the AI to build on it

```
/test-generation I have a BaseAPI class at src/apis/BaseAPI.ts.
Using it as the foundation, generate API page objects for the
following MyBasket services: UserAPI, ProductAPI, CartAPI, OrderAPI.
Each should extend BaseAPI and implement the endpoints from
http://localhost:3000/api-docs.json
```

Wait for the files to appear.

**Check each generated file and answer these questions:**

1. Does every class start with `extends BaseAPI`?
2. How does each method build its URL — hardcoded string, or `this.url(...)`?
3. How does each method handle auth — inline header, or `this.authHeader(...)`?
4. Does the constructor signature match across all four files?

**Your generated structure should look like this:**

```
src/apis/
├── BaseAPI.ts        ← your file, untouched
├── UserAPI.ts        ← extends BaseAPI
├── ProductAPI.ts     ← extends BaseAPI
├── CartAPI.ts        ← extends BaseAPI
└── OrderAPI.ts       ← extends BaseAPI
```

**What a generated file should look like:**

```typescript
// src/apis/UserAPI.ts  (AI generated)
import { BaseAPI } from './BaseAPI';

export class UserAPI extends BaseAPI {

  async register(name: string, email: string, password: string) {
    return this.request.post(this.url('/api/users/register'), {
      data: { name, email, password }
    });
  }

  async login(email: string, password: string) {
    return this.request.post(this.url('/api/users/login'), {
      data: { email, password }
    });
  }

  async getProfile(token: string) {
    return this.request.get(this.url('/api/users/profile'), {
      headers: this.authHeader(token)
    });
  }
}
```

> Notice: no hardcoded URL, no hardcoded header string.  
> The AI read `BaseAPI.ts` and extrapolated the pattern.  
> You did not write a style guide. You wrote a base class.

---

## Step 3 — Re-generate the test specs

```
/test-generation Using the page objects in src/apis/, generate
Playwright test specs for the full happy-path journey and the
edge cases from the test plan. Place specs in tests/api/
```

**Now answer the same questions you answered in Act 1:**

1. What are your test files named?
2. How is the base URL handled?
3. How is the JWT token handled?
4. How many files were generated?
5. Is there shared code between files?

**Compare your answers with your neighbour.**

---

## 🔴 Stop here — two questions

**Question 1:** What would happen if the base URL changed tomorrow?

- In your Act 1 tests:
- In your Act 2 tests:

**Question 2:** What would a new team member generate next sprint?

- With Act 1's project structure (empty):
- With Act 2's project structure (`BaseAPI.ts` present):

---

## What changed — and why

| | Act 1 | Act 2 |
|---|---|---|
| URL handling | Hardcoded in every method | `this.url()` — one place |
| Auth handling | Repeated in every method | `this.authHeader()` — inherited |
| Consistency | Varies between people | Same pattern across all files |
| New team member | Gets a random output | Gets the pattern automatically |
| Base URL change | Find and replace everywhere | Change `playwright.config.ts` once |
| Re-generation | Different output each time | Stable, predictable output |

> **The AI did not get smarter. The project got better.**  
> `BaseAPI.ts` gave the AI a contract to work within.  
> That contract is what made generation consistent.

---

## What you have at the end of Act 2

```
your-project/
├── .vscode/mcp.json
├── playwright.config.ts
├── test-plan.md
├── src/
│   └── apis/
│       ├── BaseAPI.ts          ← you wrote this
│       ├── UserAPI.ts          ← AI generated, extends BaseAPI
│       ├── ProductAPI.ts       ← AI generated, extends BaseAPI
│       ├── CartAPI.ts          ← AI generated, extends BaseAPI
│       └── OrderAPI.ts         ← AI generated, extends BaseAPI
├── tests/
│   ├── [act 1 files]
│   └── api/
│       └── [act 2 generated specs]
└── output/
    └── act1-report.html
```

---

## The question to sit with

> *Act 2 is more consistent than Act 1. But it is still only as good  
> as whoever runs the generation next.*  
>
> *What happens when the AI forgets to check `testData.ts`?  
> What happens when it generates inline data instead of using a fixture?  
> What happens when schema drift breaks an assertion and nobody knows why?*  
>
> *Consistency is not the same as governance.*

---

*Continue to [`ACT-3-governed.md`](./ACT-3-governed.md)*
