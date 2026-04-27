# Democratizing API Testing with AI
### A hands-on workshop by Raj Uppadhyay

> **"AI can generate tests in minutes. But will you trust them in three months?"**

This workshop answers that question — live, in your hands, on your machine.

---

## What you will build

A complete Playwright TypeScript API test suite for **MyBasket** — a grocery microservices application with six independent services and 22+ endpoints. You will build it three times, each time the right way getting clearer.

---

## The three acts

| File | Act | What happens |
|---|---|---|
| [`ACT-1-quick-win.md`](./ACT-1-quick-win.md) | The Quick Win | Generate a full test suite with AI in minutes — no structure, no patterns |
| [`ACT-2-structure.md`](./ACT-2-structure.md) | The Structure Problem | See why Act 1 breaks — and how one file fixes it |
| [`ACT-3-governed.md`](./ACT-3-governed.md) | The Governed Framework | Make AI generation deterministic, observable, and scalable |

> **Do not skip ahead.** Each act depends on what you discovered in the previous one.  
> Act 1 is not wrong — it is the trap. You need to fall into it first.

---

## Before you start — lab setup

Complete this before Act 1 begins.

### 1. Check prerequisites

```bash
node --version     # 18 or higher required
npm --version      # 9 or higher required
```

### 2. Install Playwright

```bash
npm init playwright@latest
# Choose: TypeScript, tests/ folder, no GitHub Actions, install browsers
```

### 3. Install the MCP server and the Skills

```bash
npx @democratize-quality/mcp-server@latest --agents
```

### 4. What this does:

- ✅ Installs the MCP server
- ✅ Sets up 4 AI-powered testing skills
- ✅ Configures VS Code integration automatically
- ✅ Creates project folders (.agents/skills/, .github/skills/, .claude/skills/, .vscode/)
- ✅ Works with: GitHub Copilot, Codex CLI, Claude Code, Cursor, and 10+ other tools

Creates `.vscode/mcp.json` in your project root:

```json
{
  "servers": {
    "democratize-quality": {
      "command": "npx",
      "args": ["@democratize-quality/mcp-server"]
    }
  }
}
```

### 5. Set up `playwright project`

Replace the contents of your `playwright.config.ts` with:

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  use: {
    baseURL: 'http://localhost:3000',
    extraHTTPHeaders: { 'Content-Type': 'application/json' },
  },
  reporter: [['html', { outputFolder: 'output' }]],
});
```

### 6. Verify your starting structure

```
your-project/
├── .vscode/
│   └── mcp.json
├── playwright.config.ts
└── tests/
    └── example.spec.ts    ← you can delete this
```

> **This is all you should have. No `src/` folder. No page objects. Nothing else.**  
> If you have more, delete it before Act 1.

### 7. Verify MyBasket is running

```bash
curl http://localhost:3000/health
# Expected: { "status": "ok" }
```

If this fails, let the presenter know before the workshop starts.

---

## The app under test — MyBasket

Six microservices. Each has its own auth requirements and its own test lifecycle.

```
API Gateway  :3000
User Service :3001    ← register, login, profile
Product Svc  :3002    ← browse, search, details
Cart Service :3003    ← add, view, remove items
Order Service:3004    ← place, view, history
AI Service   :3005    ← recommendations
```

**The user journey you will test:**

```
Register → Login → Browse Products → Add to Cart → Place Order → Verify History
```

Every step depends on the previous one. JWT tokens and resource IDs must flow between services. This dependency chain is where manual test setup quietly fails.

**API docs:** `http://localhost:3000/api-docs.json`

---

## How to use GitHub Copilot Chat with MCP

All commands in this workshop are slash commands typed into **Copilot Chat** inside VS Code.

```
/api-planning   → reads your OpenAPI schema and generates a test plan
/test-execution → runs HTTP requests and chains tokens automatically
/test-generation → converts a test plan into Playwright TypeScript tests
/test-healing   → detects schema mismatches and patches failing assertions
/init           → reads your project and generates a Copilot instructions file
```

Type these exactly as shown in each act. The MCP server handles the rest.

---

## About this workshop

**Presenter:** Raj Uppadhyay  
**MCP server:** `@democratize-quality/mcp-server`  
**Book:** *Scalable Test Automation with Playwright*  
→ bpbonline.com/products/scalable-test-automation-with-playwright

---

*Ready? Open [`ACT-1-quick-win.md`](./ACT-1-quick-win.md) and begin.*
