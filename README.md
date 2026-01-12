# API Test Framework - Best Practices & Guidelines

This document outlines the mandatory patterns and best practices for creating Playwright API tests in the Xtrem framework. Adherence to these guidelines ensures test scalability, reliability, and maintainability.

---

## Table of Contents

- [1. Getting Started](#1-getting-started)
  - [Prerequisites](#prerequisites)
  - [Install & First Run](#install--first-run)
  - [Environment Variables](#environment-variables)
- [2. Project Overview](#2-project-overview)
  - [Architecture](#architecture)
  - [Key Technologies](#key-technologies)
  - [Fixtures & Layers](#fixtures--layers)
- [3. Core Concepts](#3-core-concepts)
  - [Core Principles](#core-principles)
  - [Test Tagging Strategy](#test-tagging-strategy-priority)
  - [Test Isolation](#test-isolation-priority)
  - [Setup and Teardown](#setup-and-teardown)
  - [Data Management](#data-management)
  - [Async Operations](#async-operations)
- [4. Writing Tests](#4-writing-tests)
  - [Creating New Tests](#creating-new-tests)
  - [Creating Test Utilities](#creating-test-utilities)
  - [Common Patterns](#common-patterns)
  - [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
- [5. Running & CI](#5-running--ci)
  - [Running Tests](#running-tests-combined)
  - [CI/CD Integration](#cicd-integration-combined)
  - [Performance Guidelines](#performance-guidelines)
- [6. Troubleshooting](#6-troubleshooting-combined)
- [7. Full Code Examples](#7-full-code-examples)
  - [7.1 Setup and Teardown with Defensive Cleanup](#71-setup-and-teardown-with-defensive-cleanup)
  - [7.2 Data Factory Pattern](#72-data-factory-pattern)
  - [7.3 Async Operations – Polling with toPass](#73-async-operations--polling-with-topass)
  - [7.4 Simple CRUD Test](#74-simple-crud-test)
  - [7.5 Business Flow Test](#75-business-flow-test)
  - [7.6 Negative Testing](#76-negative-testing)
  - [7.7 Query/Filter Testing](#77-queryfilter-testing)
- [8. Best Practices](#8-best-practices)
- [9. Additional Resources](#9-additional-resources)
- [10. Quick Reference Checklist](#10-quick-reference-checklist)

---

## 1. Getting Started

### Prerequisites

- Node.js 18+ and pnpm installed
- Install dependencies with pnpm
- Application running locally or startable via Playwright

### Install & First Run

```bash
# If using a monorepo layout
cd services/api-test

# Install dependencies
pnpm install

# Run all tests
pnpm test

# Quick mode (skips layer loading)
pnpm test:quick

# Debug mode with detailed logs
pnpm test:debug

# Run only release-ready tests
pnpm test --grep "@release"

# Run only development tests
pnpm test --grep "@development"

# Exclude flaky tests (recommended for CI)
pnpm test --grep-invert "@flaky"

# Run a specific module or test file
pnpm test tests/master-data/
pnpm test tests/master-data/customer.spec.ts

# Watch mode (local development)
pnpm test --watch
```

If the app needs to be started manually (normally Playwright handles this via config):

```bash
# Start the application locally if Playwright doesn't auto-start it
pnpm start
```

### Environment Variables

- `PLAYWRIGHT_SKIP_LAYER_LOAD`: Skip loading the test data layer (faster local runs)
- `PLAYWRIGHT_GRAPH_LOG`: Enable GraphQL request/response logging
- `DEBUG`: Enable debug output (e.g., `pw:webserver`)
- `CI`: CI mode toggles retries and worker limits
- `PLAYWRIGHT_SHARD_TOTAL` / `PLAYWRIGHT_SHARD_CURRENT`: CI sharding controls

Examples:

```bash
# POSIX
PLAYWRIGHT_GRAPH_LOG=true PLAYWRIGHT_SKIP_LAYER_LOAD=true pnpm test
DEBUG=pw:webserver PLAYWRIGHT_GRAPH_LOG=true pnpm test

# Windows PowerShell
$env:PLAYWRIGHT_GRAPH_LOG = "true"; pnpm test
$env:DEBUG = "pw:webserver"; pnpm test
```

---

## 2. Project Overview

### Architecture

```
services/api-test/
├── lib/
│   ├── index.ts                      # Typed test fixture export
│   ├── functions/
│   │   └── random.ts                 # Random data generation utilities
│   ├── master-data/
│   │   ├── business-entity.ts
│   │   ├── customer.ts
│   │   ├── item.ts
│   │   ├── site.ts
│   │   ├── supplier.ts
│   │   └── index.ts
│   ├── sales/
│   │   ├── sales-order.ts
│   │   └── index.ts
│   ├── system/
│   │   └── index.ts
│   └── reporters/
│       └── error-summary-reporter.ts # Custom error reporting
├── tests/
│   ├── master-data/
│   ├── sales/
│   └── system/
├── playwright.config.ts
├── package.json
├── tsconfig.json
└── setup.ts                          # Global setup (data layer loading)
```

### Key Technologies

- Playwright: API testing support with rich tooling
- TypeScript: Typed GraphQL interactions
- `@sage/xtrem-playwright`: Framework utilities, fixtures, base classes
- `@sage/xtrem-client`: Typed GraphQL client
- BaseNode: Generic CRUD wrapper for GraphQL nodes

### Fixtures & Layers

Fixtures (from lib/index.ts):

```typescript
// Export a strongly-typed Playwright test fixture bound to the Graph API
// This provides typed `graph` (queries) and `graphMutation` (mutations)
export const test: XtremTestType<GraphApi> = createXtremTest<GraphApi>();
```

- `graph`: Typed read-only GraphQL queries
- `graphMutation`: Typed mutations/write operations

Helper classes extend `BaseNode` and provide:
- `query()`, `create()`, `update()`, `delete()`, `queryRandom()`
- Strongly typed selectors and random data generation helpers

---

## 3. Core Concepts

### Core Principles

1. Isolation: Tests run independently, in any order, any time
2. Atomicity: Each test creates and cleans up its own data
3. Determinism: Consistent results across environments

Why this matters:
- Enables efficient parallelization and sharding
- Prevents cascading failures
- Reduces maintenance and debugging effort

### Test Tagging Strategy

- `@release`: Stable, production-ready (runs in main pipeline)
- `@development`: In progress/verification (runs in dev pipeline)
- `@flaky`: Temporarily unstable (excluded from main pipeline; must be remediated within 1 sprint)

Rules:
- Every test must have exactly one of these tags
- Combine suites via pipe when needed: `@development|@release`

Run examples:

```bash
pnpm test --grep "@release"
pnpm test --grep "@development"
pnpm test --grep "@flaky"
pnpm test --grep-invert "@flaky"
```

### Test Isolation

Zero-shared state:
- No reliance on other tests or pre-seeded records
- Use unique identifiers (timestamps/UUIDs)
- Ensure tests can run alone and in any order

Self-check:
- Runs alone with `pnpm test tests/path/to/test.spec.ts`
- Creates all required data programmatically
- Cleans up all created data
- Uses unique identifiers

### Setup and Teardown

Mandatory cleanup pattern:
- Defensive deletion in `afterEach`
- Delete children before parents
- Wrap deletions in `.catch()` and guard with `if (entity?._id)`
- Track created IDs in test scope for reliable cleanup

### Data Management

- Never hard-code IDs or depend on manual seeds
- Create all dependencies within the test or `beforeEach`
- Prefer helper classes/factories for uniform creation
- Increase randomness or add timestamps to avoid collisions

### Async Operations

Use polling only for genuinely async processes (allocations, jobs, webhooks):
- `expect(async fn).toPass({ timeout })` with explicit timeouts
- Poll the source of truth by reloading the entity inside the loop
- Do not poll synchronous CRUD; assert directly

Timeout guidelines:
- Quick: ~5s
- Standard: ~30s
- Long-running: ~60s

---

## 4. Writing Tests

### Creating New Tests

Steps:
1. Identify domain (master-data, sales, system)
2. Create `.spec.ts` in the appropriate folder
3. Use `test` fixture (`graph`/`graphMutation`) for typed calls
4. Organize with `test.step()` for clarity
5. Add exactly one tag: `@release`, `@development`, or `@flaky`

Basic scaffold:

```typescript
// Playwright's assertion library
import { expect } from '@playwright/test';
// Project-specific typed test fixture that exposes Graph clients
import { test } from '../../lib';
// Domain entry point for master-data helpers (e.g., `Customer`, `Item`)
import { MasterData } from '../../lib/master-data';

test.describe('Partner', () => {
  // Each test must have exactly one tag (here: @development)
  // The fixture parameter provides typed clients: `{ graph, graphMutation }`
  test('@development Partner CRUD operations', async ({ graph }) => {
    // ...
  });
});
```

### Creating Test Utilities

Recommended patterns:
- Domain helper classes (extend `BaseNode`) for selectors + CRUD + random data
- Pure utility helpers (date, formatting, randoms, assertions)
- Setup fixtures for common datasets per suite
- Assertion helpers for complex validations
- Fluent builders for complex inputs

### Common Patterns

- Simple CRUD with `afterEach` cleanup
- Business flows that create prerequisites, then clean up in reverse
- Negative/validation tests using `rejects.toThrow`
- Query/filter tests using selectors and filters

### Anti-Patterns to Avoid

- Shared global state between tests
- Missing cleanup
- Hard-coded dependencies (IDs, seeded data)
- Serial tests that force order (`test.describe.serial`)
- Missing mandatory tags
- Polling synchronous CRUD

---

## 5. Running & CI

### Running Tests

```bash
# Ensure dependencies
pnpm install

# Start services if required
pnpm start

# Run tests
pnpm test

# With logging
PLAYWRIGHT_GRAPH_LOG=true pnpm test

# Single file or directory
pnpm test tests/master-data/customer.spec.ts
pnpm test tests/master-data/

# Tag-based
pnpm test --grep "@development"
pnpm test --grep "@release"

# Watch mode
pnpm test --watch

# Workers and sharding
pnpm test --workers=4
pnpm test --workers=100%
pnpm test --shard=1/4
```

### CI/CD Integration

- Fully parallel execution (`fullyParallel: true`) for optimal distribution
- Shard tests across agents with env vars or CLI flags
- Retries enabled in CI for transient failures; fewer/no retries locally
- Limit workers in CI for resource efficiency (e.g., 3)
- Generate HTML reports; optionally capture traces only on first retry

Report merging example:

```bash
npx playwright merge-reports --reporter html blob-report-*
```

### Performance Guidelines

Standard timeouts:

```typescript
const ALLOCATION_TIMEOUT = 60000;  // 60s – long-running jobs
const STANDARD_TIMEOUT   = 30000;  // 30s – typical async
const QUICK_TIMEOUT      = 5000;   // 5s  – quick checks
```

Optimization tips:
- Use `test:quick` during development
- Run specific files instead of the entire suite
- Limit local workers (`--workers=2`)
- Use `--grep` for subsets
- Provision only the data you need

---

## 6. Troubleshooting

- Cannot query field: Fix selector to match GraphQL schema/types
- "No _id to update": Create the entity or pass its ID before updating
- Slow layer load: Use quick mode (`PLAYWRIGHT_SKIP_LAYER_LOAD=true`)
- Passes locally, fails in CI: Examine timing, environment parity, parallel conflicts; temporarily mark `@flaky` with a remediation plan
- Empty query result: Enable Graph logging, validate filters match created data
- Random conflicts: Increase entropy or include timestamps

---

## 7. Full Code Examples

These examples embody isolation, defensive cleanup, factories, async polling, and common patterns.

### 7.1 Setup and Teardown with Defensive Cleanup

```typescript
// Import Playwright assertions
import { expect } from '@playwright/test';
// Import the typed test fixture and domain helpers
import { test } from '../../lib';
import { SalesOrder, SalesShipment, SalesInvoice, Customer, Site, Item } from '@lib';

test.describe('Sales Order Tests (Setup & Teardown)', () => {
  // Test-local registry of entities we create to ensure deterministic cleanup
  const testData: {
    customer?: any;
    site?: any;
    item?: any;
    order?: any;
    shipment?: any;
    invoice?: any;
  } = {};

  test.beforeEach(async ({ graph }) => {
    // Create required dependencies up-front; add randomness to avoid collisions
    testData.customer = await new Customer(graph).create({
      name: `Customer-${Date.now()}`,
      email: `test-${Date.now()}@example.com`,
      isActive: true,
    });
    // Create a site used by the order
    testData.site = await new Site(graph).create({ name: `Site-${Date.now()}`, isActive: true });
    // Create an item we'll sell on the order
    testData.item = await new Item(graph).create({ name: `Item-${Date.now()}`, price: 100, isActive: true });
  });

  test.afterEach(async ({ graph }) => {
    // Defensive cleanup: delete children before parents, each wrapped in catch
    const tasks: Array<Promise<any>> = [];
    if (testData.invoice?._id) tasks.push(new SalesInvoice(graph).delete(testData.invoice._id).catch(() => {}));
    if (testData.shipment?._id) tasks.push(new SalesShipment(graph).delete(testData.shipment._id).catch(() => {}));
    if (testData.order?._id) tasks.push(new SalesOrder(graph).delete(testData.order._id).catch(() => {}));
    if (testData.item?._id) tasks.push(new Item(graph).delete(testData.item._id).catch(() => {}));
    if (testData.site?._id) tasks.push(new Site(graph).delete(testData.site._id).catch(() => {}));
    if (testData.customer?._id) tasks.push(new Customer(graph).delete(testData.customer._id).catch(() => {}));
    await Promise.all(tasks);
  });

  test('@release Create sales order', async ({ graph }) => {
    // Create an order that references our freshly-created dependencies
    testData.order = await new SalesOrder(graph).create({
      customer: testData.customer._id,
      site: testData.site._id,
      items: [{ item: testData.item._id, quantity: 5 }],
    });
    // Validate the order was created and wired correctly
    expect(testData.order._id).toBeDefined();
    expect(testData.order.customer).toBe(testData.customer._id);
  });
});
```

### 7.2 Data Factory Pattern

```typescript
// lib/test-utils/factories.ts
import { Customer, Site, Item, SalesOrder } from '@lib';

// Create a randomized, valid customer for tests
export async function createTestCustomer(graph: any, overrides: Record<string, any> = {}) {
  return new Customer(graph).create({
    name: `Customer-${Date.now()}`,
    email: `test-${Date.now()}@example.com`,
    isActive: true,
    ...overrides,
  });
}

// Create a randomized, valid site for tests
export async function createTestSite(graph: any, overrides: Record<string, any> = {}) {
  return new Site(graph).create({ name: `Site-${Date.now()}`, isActive: true, ...overrides });
}

// Create a randomized, valid item for tests
export async function createTestItem(graph: any, overrides: Record<string, any> = {}) {
  return new Item(graph).create({ name: `Item-${Date.now()}`, price: 100, isActive: true, ...overrides });
}

// Compose an order with auto-provisioned dependencies; dependencies can be injected
export async function createTestSalesOrder(graph: any, deps: Partial<{ customer: any; site: any; item: any }> = {}) {
  const customer = deps.customer || await createTestCustomer(graph);
  const site = deps.site || await createTestSite(graph);
  const item = deps.item || await createTestItem(graph);

  const order = await new SalesOrder(graph).create({
    customer: customer._id,
    site: site._id,
    items: [{ item: item._id, quantity: 10 }],
  });

  // Return both the created order and the dependencies for optional cleanup
  return { order, dependencies: { customer, site, item } };
}

// Usage in tests
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { createTestSalesOrder } from '../../lib/test-utils/factories';

test('@release Order with factory', async ({ graph }) => {
  // One-liner to create an order with valid dependencies
  const { order } = await createTestSalesOrder(graph);
  expect(order._id).toBeDefined();
});
```

### 7.3 Async Operations – Polling with toPass

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { SalesOrder } from '@lib';

// Explicit timeout buckets for different async categories
const ALLOCATION_TIMEOUT = 60000; // 60s – long-running async
const STANDARD_TIMEOUT = 30000;   // 30s – typical async
const QUICK_TIMEOUT = 5000;       // 5s  – status checks

test('@release Verify allocation', async ({ graph }) => {
  // Create and confirm an order, then poll the source of truth
  const orders = new SalesOrder(graph);
  const order = await orders.create({ /* deps provisioned in beforeEach */ });
  await orders.confirm(order._id);

  // Poll using expect(...).toPass with a clear timeout; re-read the entity
  await expect(async () => {
    const refreshed = await orders.get(order._id);
    expect(refreshed.allocatedQuantity).toBeGreaterThan(0);
  }).toPass({ timeout: ALLOCATION_TIMEOUT });
});

test('@release CRUD remains synchronous', async ({ graph }) => {
  // For synchronous operations, assert directly without polling
  const orders = new SalesOrder(graph);
  const order = await orders.create({ /* deps provisioned in beforeEach */ });
  const updated = await orders.update(order._id, { comment: 'Updated' });
  expect(updated.comment).toBe('Updated');
});
```

### 7.4 Simple CRUD Test

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { Customer } from '@lib';

test.describe('Customer CRUD Operations', () => {
  // Store the created entity per-test for cleanup
  let created: any;

  test.afterEach(async ({ graph }) => {
    // Defensive delete: guard existence and catch errors
    if (created?._id) {
      await new Customer(graph).delete(created._id).catch(() => {});
      created = null;
    }
  });

  test('@release Create customer with valid data', async ({ graph }) => {
    // Create a valid customer and assert key fields
    created = await new Customer(graph).create({
      name: `Customer-${Date.now()}`,
      email: `test-${Date.now()}@example.com`,
      isActive: true,
    });
    expect(created._id).toBeDefined();
    expect(created.email).toContain('@example.com');
  });

  test('@release Update customer name', async ({ graph }) => {
    // Update a field and validate the result is persisted
    created = await new Customer(graph).create({ name: `C-${Date.now()}`, email: `t-${Date.now()}@example.com` });
    const updated = await new Customer(graph).update(created._id, { name: 'Updated Name' });
    expect(updated.name).toBe('Updated Name');
  });

  test('@release Delete customer', async ({ graph }) => {
    // Delete and rely on afterEach to avoid orphaned state
    created = await new Customer(graph).create({ name: `C-${Date.now()}`, email: `t-${Date.now()}@example.com` });
    await new Customer(graph).delete(created._id);
    created = null;
  });
});
```

### 7.5 Business Flow Test

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { SalesOrder, SalesShipment, SalesInvoice, Customer, Site, Item } from '@lib';

test.describe('Sales Business Flow', () => {
  // Track all created entities to clean up in reverse order
  const testData: { customer?: any; site?: any; item?: any; order?: any; shipment?: any; invoice?: any } = {};

  test.beforeEach(async ({ graph }) => {
    // Minimal prerequisite provisioning for the flow
    testData.customer = await new Customer(graph).create({ name: `Cust-${Date.now()}`, email: `t-${Date.now()}@example.com` });
    testData.site = await new Site(graph).create({ name: `Site-${Date.now()}` });
    testData.item = await new Item(graph).create({ name: `Item-${Date.now()}`, price: 10 });
  });

  test.afterEach(async ({ graph }) => {
    // Cleanup pipeline: children first, each deletion guarded
    const cleaners: Array<Promise<any>> = [];
    if (testData.invoice?._id) cleaners.push(new SalesInvoice(graph).delete(testData.invoice._id).catch(() => {}));
    if (testData.shipment?._id) cleaners.push(new SalesShipment(graph).delete(testData.shipment._id).catch(() => {}));
    if (testData.order?._id) cleaners.push(new SalesOrder(graph).delete(testData.order._id).catch(() => {}));
    if (testData.item?._id) cleaners.push(new Item(graph).delete(testData.item._id).catch(() => {}));
    if (testData.site?._id) cleaners.push(new Site(graph).delete(testData.site._id).catch(() => {}));
    if (testData.customer?._id) cleaners.push(new Customer(graph).delete(testData.customer._id).catch(() => {}));
    await Promise.all(cleaners);
  });

  test('@release Complete flow: Order → Ship → Invoice', async ({ graph }) => {
    // Create order, then ship and invoice it; validate the end result
    const orders = new SalesOrder(graph);
    const shipments = new SalesShipment(graph);
    const invoices = new SalesInvoice(graph);

    testData.order = await orders.create({
      customer: testData.customer._id,
      site: testData.site._id,
      items: [{ item: testData.item._id, quantity: 3 }],
    });

    testData.shipment = await shipments.create({ order: testData.order._id });
    testData.invoice = await invoices.create({ order: testData.order._id });

    expect(testData.invoice._id).toBeDefined();
  });
});
```

### 7.6 Negative Testing

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { SalesOrder, Customer } from '@lib';

test.describe('Sales Order Validation', () => {
  // A valid customer used in negative scenarios
  let testCustomer: any;

  test.beforeEach(async ({ graph }) => {
    testCustomer = await new Customer(graph).create({ name: `Cust-${Date.now()}`, email: `t-${Date.now()}@example.com` });
  });

  test.afterEach(async ({ graph }) => {
    // Defensive cleanup for the seeded customer
    if (testCustomer?._id) await new Customer(graph).delete(testCustomer._id).catch(() => {});
  });

  test('@release Reject order with negative quantity', async ({ graph }) => {
    // Expect the create call to throw due to invalid quantity
    await expect(async () => {
      await new SalesOrder(graph).create({ customer: testCustomer._id, items: [{ item: 'invalid', quantity: -1 }] });
    }).rejects.toThrow(/quantity must be positive/i);
  });

  test('@release Reject order without customer', async ({ graph }) => {
    // Missing required `customer` should produce a validation error
    await expect(async () => {
      await new SalesOrder(graph).create({ items: [{ item: 'x', quantity: 1 }] });
    }).rejects.toThrow(/customer is required/i);
  });

  test('@release Reject order with invalid item', async ({ graph }) => {
    // Invalid item reference should fail
    await expect(async () => {
      await new SalesOrder(graph).create({ customer: testCustomer._id, items: [{ item: '#DOES_NOT_EXIST', quantity: 1 }] });
    }).rejects.toThrow(/item not found/i);
  });
});
```

### 7.7 Query/Filter Testing

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { Item } from '@lib';

test.describe('Item Master Data Query', () => {
  // Keep references to created items for cleanup after each test
  const createdItems: any[] = [];

  test.beforeEach(async ({ graph }) => {
    // Seed a small variety of items for filter-based queries
    const items = [
      { name: `A-${Date.now()}`, price: 10, isActive: true },
      { name: `B-${Date.now()}`, price: 50, isActive: false },
      { name: `C-${Date.now()}`, price: 100, isActive: true },
    ];
    for (const data of items) createdItems.push(await new Item(graph).create(data));
  });

  test.afterEach(async ({ graph }) => {
    // Clean all created items; use splice to consume the array
    for (const i of createdItems.splice(0)) await new Item(graph).delete(i._id).catch(() => {});
  });

  test('@release Query items with filters', async ({ graph }) => {
    // Query with combined filters and a selector to limit returned fields
    const items = await new Item(graph).query({
      filter: { isActive: { eq: true }, price: { gte: 50 } },
      selector: { _id: true, name: true, price: true, isActive: true },
      limit: 100,
    });
    // Validate all results match the filter criteria
    expect(items.length).toBeGreaterThan(0);
    for (const it of items) {
      expect(it.isActive).toBe(true);
      expect(it.price).toBeGreaterThanOrEqual(50);
    }
  });
});
```

---

## 8. Best Practices

- Test independence: no shared state; own data lifecycle
- Use `test.step()` to structure complex flows
- Write meaningful assertions with clear messages
- Prefer domain helper classes over raw graph calls
- Always clean up created data (especially for `@release`)
- Use appropriate tags to control CI behavior
- Handle random data properly; store IDs for follow-up queries
- Organize tests by business domain

---

## 9. Additional Resources

- Playwright Docs (API Testing): https://playwright.dev/docs/api-testing
- `@sage/xtrem-playwright`: Framework utilities and fixtures
- `@sage/xtrem-client`: GraphQL client and types
- BaseNode reference: typically `platform/testing/xtrem-playwright/lib/base-node.ts`

---

## 10. Quick Reference Checklist

- [ ] Exactly one tag: `@release`, `@development`, or `@flaky`
- [ ] Programmatically creates all required data
- [ ] `afterEach` performs defensive cleanup in reverse order
- [ ] Cleanup guards existence and catches errors
- [ ] No hard-coded IDs or seeded dependencies
- [ ] Runs independently in isolation
- [ ] Uses unique identifiers (timestamps/UUIDs)
- [ ] Async uses `toPass()` with explicit timeouts (only when truly async)
- [ ] Naming: `@tag Description`
- [ ] Deletes children first, then parents
