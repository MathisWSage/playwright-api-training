# API Test Framework - Best Practices & Guidelines

This document outlines the mandatory patterns and best practices for creating Playwright API tests in the Xtrem framework. Adherence to these guidelines ensures test scalability, reliability, and maintainability.

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Core Principles](#core-principles)
3. [Test Isolation](#test-isolation)
4. [Setup and Teardown](#setup-and-teardown)
5. [Test Tagging Strategy](#test-tagging-strategy)
6. [Data Management](#data-management)
7. [Async Operations](#async-operations)
8. [Common Patterns](#common-patterns)
9. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
10. [Example Test Templates](#example-test-templates)
11. [Running Tests](#running-tests)
12. [CI/CD Integration](#cicd-integration)

---

## Quick Start

### Available Commands

```bash
# Standard test launcher
pnpm test

# Skip layer loading for faster execution
pnpm test:quick

# Add detailed logs for debugging
pnpm test:debug

# Run only release-ready tests
pnpm test --grep "@release"

# Run all tests except flaky ones
pnpm test --grep-invert "@flaky"
```

### Environment Variables

- `PLAYWRIGHT_GRAPH_LOG=true` - Log all GraphQL queries and results
- `DEBUG=pw:webserver` - Show server logs if launched by Playwright
- `PLAYWRIGHT_SKIP_LAYER_LOAD=true` - Skip layer loading (same as `test:quick`)

---

## Core Principles

### The Three Pillars of Reliable API Tests

1. **Isolation**: Every test must run independently, in any order, at any time
2. **Atomicity**: Each test must create and clean up its own data
3. **Determinism**: Tests must produce consistent results across all environments

**Why This Matters**:
- Enables efficient parallelization and sharding
- Prevents "domino effect" cascading failures
- Reduces maintenance overhead and debugging time
- Builds trust in the test suite

---

## Test Isolation

### Zero-Shared State Architecture

**RULE**: Tests must NEVER depend on data created by other tests or assume pre-existing database records.

```typescript
// ❌ BAD - Relies on pre-existing data
test('Update customer', async ({ graph }) => {
    const customer = await customerClass.update('#DE072', { name: 'Updated' });
    expect(customer.name).toBe('Updated');
});

// ✅ GOOD - Creates own data
test('Update customer', async ({ graph }) => {
    // Provision
    const customer = await customerClass.create({
        name: `Test-Customer-${Date.now()}`,
        email: `test-${Date.now()}@example.com`,
    });

    // Test
    const updated = await customerClass.update(customer._id, { name: 'Updated' });

    // Assert
    expect(updated.name).toBe('Updated');

    // Cleanup happens in afterEach
});
```

### Test Independence Checklist

Before committing a test, verify:

- ✅ Can this test run alone? (`pnpm test tests/path/to/test.spec.ts`)
- ✅ Can it run in any order with other tests?
- ✅ Does it create all required data programmatically?
- ✅ Does it clean up all created data?
- ✅ Does it use unique identifiers (timestamps, UUIDs)?

---

## Setup and Teardown

### Mandatory Cleanup Pattern

**RULE**: Every test that creates data MUST implement an `afterEach` hook to delete it.

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { SalesOrder, Customer, Site, Item } from '@lib';

test.describe('Sales Order Tests', () => {
    // Storage for created resources
    const testData: {
        customer?: any;
        site?: any;
        item?: any;
        order?: any;
    } = {};

    test.beforeEach(async ({ graph }) => {
        // Programmatically provision all dependencies
        testData.customer = await new Customer(graph).create({
            name: `Customer-${Date.now()}`,
            email: `test-${Date.now()}@example.com`,
        });

        testData.site = await new Site(graph).create({
            name: `Site-${Date.now()}`,
        });

        testData.item = await new Item(graph).create({
            name: `Item-${Date.now()}`,
            price: 100,
        });
    });

    test.afterEach(async ({ graph }) => {
        // Defensive cleanup - order matters (delete children first)
        const cleanupOrder = [
            { id: testData.order?._id, fn: () => SalesOrder.delete(testData.order._id) },
            { id: testData.item?._id, fn: () => Item.delete(testData.item._id) },
            { id: testData.site?._id, fn: () => Site.delete(testData.site._id) },
            { id: testData.customer?._id, fn: () => Customer.delete(testData.customer._id) },
        ];

        for (const resource of cleanupOrder) {
            if (resource.id) {
                await resource.fn().catch((error) => {
                    // Log but don't fail on cleanup errors
                    console.warn(`Cleanup failed for ${resource.id}:`, error.message);
                });
            }
        }
    });

    test('@release Create sales order', async ({ graph }) => {
        testData.order = await new SalesOrder(graph).create({
            customer: testData.customer._id,
            site: testData.site._id,
            items: [{ item: testData.item._id, quantity: 5 }],
        });

        expect(testData.order._id).toBeDefined();
        expect(testData.order.customer).toBe(testData.customer._id);
    });
});
```

### Cleanup Best Practices

1. **Use defensive deletion**: Wrap deletes in `.catch()` to prevent secondary failures
2. **Delete in reverse order**: Children before parents (invoices before orders)
3. **Check for existence**: Only delete if the resource was created (`if (resource?._id)`)
4. **Store IDs in test scope**: Use a `testData` object to track created resources
5. **Handle 404s gracefully**: Deletion should not fail if resource doesn't exist

---

## Test Tagging Strategy

### Mandatory Tags

**RULE**: Every test MUST have exactly ONE of these tags in its description:

| Tag | Purpose | CI Behavior | SLA |
|-----|---------|-------------|-----|
| `@release` | Stable, production-ready tests | ✅ Runs in main pipeline | N/A |
| `@development` | Tests under development/verification | ⚠️ Runs in dev pipeline only | N/A |
| `@flaky` | Known unstable tests (temporary) | ❌ Excluded from main pipeline | Max 1 Sprint |

### Tagging Examples

```typescript
// Stable test ready for production
test('@release Complete sales order flow', async ({ graph }) => {
    // test body
});

// Test being developed/stabilized
test('@development New allocation feature', async ({ graph }) => {
    // test body
});

// Temporarily flaky - MUST be fixed within 1 sprint
test('@flaky Intermittent timeout issue', async ({ graph }) => {
    // test body
});
```

### Flaky Test Management

#### Quarantine Policy

- **Maximum quarantine time**: 1 Sprint
- **Priority**: Equivalent to P2 Product Defect
- **Escalation**: After 1 sprint → Engineering Manager
- **Dashboard**: Flaky test count must be visible on main dashboard

#### Remediation Process

1. Tag test with `@flaky`
2. Create ticket with root cause analysis
3. Schedule fix within sprint
4. Prove stability (50 consecutive passes)
5. Remove `@flaky` tag

**WARNING**: Aggressive tagging of tests as `@flaky` to "clean the board" is prohibited. This represents a blind spot in quality gates.

---

## Data Management

### Programmatic Provisioning Pattern

**RULE**: Never rely on hard-coded entity IDs or manual database seeds.

```typescript
// ❌ BAD - Hard-coded dependencies
test('Create order', async ({ graph }) => {
    const order = await salesOrderClass.create({
        customer: '#DE072',        // Assumes this exists
        site: '#D1S',              // Assumes this exists
        item: '#ID-091210',        // Assumes this exists
    });
});

// ✅ GOOD - Programmatic provisioning
test('@release Create order', async ({ graph }) => {
    // Create all dependencies
    const customer = await new Customer(graph).create({
        name: `Test-${Date.now()}`,
        email: `test-${Date.now()}@example.com`,
    });

    const site = await new Site(graph).create({
        name: `Site-${Date.now()}`,
    });

    const item = await new Item(graph).create({
        name: `Item-${Date.now()}`,
        price: 100,
    });

    // Use programmatically created IDs
    const order = await salesOrderClass.create({
        customer: customer._id,
        site: site._id,
        items: [{ item: item._id, quantity: 10 }],
    });

    expect(order._id).toBeDefined();

    // Cleanup in afterEach
});
```

### Why Avoid Hard-coded Dependencies?

1. **Environment coupling**: Test fails if `#DE072` doesn't exist or is modified
2. **Parallel execution issues**: Multiple tests modifying the same entity
3. **Data rot**: Pre-seeded data degrades over time
4. **Debugging difficulty**: Failures don't indicate which dependency is missing

### Data Factory Pattern (Advanced)

For complex data setups, create factory functions:

```typescript
// lib/test-utils/factories.ts
export async function createTestCustomer(graph: any, overrides = {}) {
    return await new Customer(graph).create({
        name: `Customer-${Date.now()}`,
        email: `test-${Date.now()}@example.com`,
        ...overrides,
    });
}

export async function createTestSalesOrder(graph: any, dependencies = {}) {
    const customer = dependencies.customer || await createTestCustomer(graph);
    const site = dependencies.site || await createTestSite(graph);
    const item = dependencies.item || await createTestItem(graph);

    return {
        order: await new SalesOrder(graph).create({
            customer: customer._id,
            site: site._id,
            items: [{ item: item._id, quantity: 10 }],
        }),
        dependencies: { customer, site, item },
    };
}

// Usage in tests
test('@release Order with factory', async ({ graph }) => {
    const { order, dependencies } = await createTestSalesOrder(graph);

    expect(order._id).toBeDefined();

    // Cleanup dependencies in afterEach
});
```

---

## Async Operations

### Polling Pattern for Async Processes

Some operations (like allocation, background jobs) are genuinely asynchronous. Use Playwright's `toPass()` with explicit timeouts:

```typescript
const ALLOCATION_TIMEOUT = 60000; // 60 seconds
const STANDARD_TIMEOUT = 30000;   // 30 seconds
const QUICK_TIMEOUT = 5000;       // 5 seconds

test('@release Verify allocation', async ({ graph }) => {
    // Create and confirm order
    const order = await createAndConfirmOrder(graph);

    // Poll for allocation completion
    await expect(async () => {
        const updated = await salesOrderClass.get(order._id);
        expect(updated.allocationStatus).toBe('Allocated');
        expect(updated.allocatedQuantity).toBeGreaterThan(0);
    }).toPass({ timeout: ALLOCATION_TIMEOUT });
});
```

### Async Operation Guidelines

1. **Use `toPass()` only for genuinely async operations** (allocation, webhooks, background jobs)
2. **Set explicit timeouts** based on operation type:
   - Quick operations (status updates): 5 seconds
   - Standard operations: 30 seconds
   - Long-running operations (allocation): 60 seconds
3. **Poll the source of truth**: Reload entity, don't rely on cached data
4. **Fail fast for synchronous operations**: Don't poll for CRUD operations

### When NOT to Use Polling

```typescript
// ❌ BAD - Polling synchronous CRUD operation
await expect(async () => {
    const customer = await customerClass.get(customerId);
    expect(customer.name).toBe('Test');
}).toPass();

// ✅ GOOD - Direct assertion for synchronous operation
const customer = await customerClass.get(customerId);
expect(customer.name).toBe('Test');
```

---

## Common Patterns

### Pattern 1: Simple CRUD Test

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { Customer } from '@lib';

test.describe('Customer Management', () => {
    let customer: any;

    test.afterEach(async ({ graph }) => {
        if (customer?._id) {
            await Customer.delete(customer._id).catch(() => {});
        }
    });

    test('@release Create customer with valid data', async ({ graph }) => {
        customer = await new Customer(graph).create({
            name: `Customer-${Date.now()}`,
            email: `test-${Date.now()}@example.com`,
        });

        expect(customer._id).toBeDefined();
        expect(customer.name).toContain('Customer-');
        expect(customer.email).toContain('@example.com');
    });

    test('@release Update customer', async ({ graph }) => {
        customer = await new Customer(graph).create({
            name: 'Original Name',
        });

        const updated = await Customer.update(customer._id, {
            name: 'Updated Name',
        });

        expect(updated.name).toBe('Updated Name');
    });

    test('@release Delete customer', async ({ graph }) => {
        customer = await new Customer(graph).create({
            name: 'To Delete',
        });

        await Customer.delete(customer._id);

        await expect(Customer.get(customer._id)).rejects.toThrow();
        customer = null; // Prevent double-delete in afterEach
    });
});
```

### Pattern 2: Business Flow Test

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import {
    SalesOrder,
    SalesShipment,
    SalesInvoice,
    Customer,
    Site,
    Item,
} from '@lib';

test.describe('Sales Business Flow', () => {
    const testData: {
        customer?: any;
        site?: any;
        item?: any;
        order?: any;
        shipment?: any;
        invoice?: any;
    } = {};

    test.beforeEach(async ({ graph }) => {
        testData.customer = await new Customer(graph).create({
            name: `Customer-${Date.now()}`,
        });
        testData.site = await new Site(graph).create({
            name: `Site-${Date.now()}`,
        });
        testData.item = await new Item(graph).create({
            name: `Item-${Date.now()}`,
            price: 100,
        });
    });

    test.afterEach(async ({ graph }) => {
        const cleanup = [
            { id: testData.invoice?._id, fn: () => SalesInvoice.delete(testData.invoice._id) },
            { id: testData.shipment?._id, fn: () => SalesShipment.delete(testData.shipment._id) },
            { id: testData.order?._id, fn: () => SalesOrder.delete(testData.order._id) },
            { id: testData.item?._id, fn: () => Item.delete(testData.item._id) },
            { id: testData.site?._id, fn: () => Site.delete(testData.site._id) },
            { id: testData.customer?._id, fn: () => Customer.delete(testData.customer._id) },
        ];

        for (const resource of cleanup) {
            if (resource.id) {
                await resource.fn().catch(() => {});
            }
        }
    });

    test('@release Complete flow: Order → Ship → Invoice', async ({ graph }) => {
        // Step 1: Create order
        testData.order = await new SalesOrder(graph).create({
            customer: testData.customer._id,
            site: testData.site._id,
            items: [{ item: testData.item._id, quantity: 10 }],
        });
        expect(testData.order.status).toBe('Draft');

        // Step 2: Confirm order
        await salesOrderClass.confirm(testData.order._id);
        testData.order = await salesOrderClass.get(testData.order._id);
        expect(testData.order.status).toBe('Confirmed');

        // Step 3: Create shipment
        testData.shipment = await new SalesShipment(graph).create({
            order: testData.order._id,
        });
        expect(testData.shipment._id).toBeDefined();

        // Step 4: Create invoice
        testData.invoice = await new SalesInvoice(graph).create({
            order: testData.order._id,
        });
        expect(testData.invoice._id).toBeDefined();
    });
});
```

### Pattern 3: Negative Testing

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { SalesOrder, Customer } from '@lib';

test.describe('Sales Order Validation', () => {
    let testCustomer: any;

    test.beforeEach(async ({ graph }) => {
        testCustomer = await new Customer(graph).create({
            name: `Customer-${Date.now()}`,
        });
    });

    test.afterEach(async ({ graph }) => {
        if (testCustomer?._id) {
            await Customer.delete(testCustomer._id).catch(() => {});
        }
    });

    test('@release Reject order with negative quantity', async ({ graph }) => {
        await expect(
            new SalesOrder(graph).create({
                customer: testCustomer._id,
                items: [{ item: 'ITEM-001', quantity: -5 }],
            })
        ).rejects.toThrow(/quantity must be positive/i);
    });

    test('@release Reject order without customer', async ({ graph }) => {
        await expect(
            new SalesOrder(graph).create({
                items: [{ item: 'ITEM-001', quantity: 5 }],
            })
        ).rejects.toThrow(/customer is required/i);
    });

    test('@release Reject order with invalid item', async ({ graph }) => {
        await expect(
            new SalesOrder(graph).create({
                customer: testCustomer._id,
                items: [{ item: 'NONEXISTENT', quantity: 5 }],
            })
        ).rejects.toThrow(/item not found/i);
    });
});
```

### Pattern 4: Query/Filter Testing

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { Item, BaseQuery } from '@lib';

test.describe('Item Master Data Query', () => {
    const createdItems: any[] = [];

    test.beforeEach(async ({ graph }) => {
        // Create test dataset
        const categories = ['Electronics', 'Furniture', 'Supplies'];
        for (const category of categories) {
            const item = await new Item(graph).create({
                name: `Item-${category}-${Date.now()}`,
                category,
                price: Math.random() * 1000,
            });
            createdItems.push(item);
        }
    });

    test.afterEach(async ({ graph }) => {
        for (const item of createdItems) {
            await Item.delete(item._id).catch(() => {});
        }
        createdItems.length = 0;
    });

    test('@release Query items with filters', async ({ graph }) => {
        const query = new BaseQuery(graph, 'Item');
        const results = await query.filter({
            category: 'Electronics',
        }).execute();

        expect(results.length).toBeGreaterThan(0);
        results.forEach((item) => {
            expect(item.category).toBe('Electronics');
        });
    });
});
```

---

## Anti-Patterns to Avoid

### ❌ Anti-Pattern 1: Shared Global State

```typescript
// BAD - Tests depend on execution order
let globalCustomer: any;

test('Create customer', async ({ graph }) => {
    globalCustomer = await new Customer(graph).create({ name: 'Test' });
});

test('Update customer', async ({ graph }) => {
    // Fails if previous test didn't run or failed
    await Customer.update(globalCustomer._id, { name: 'Updated' });
});
```

**Why It's Bad**: Tests cannot run independently. Parallel execution fails. Debugging is difficult.

### ❌ Anti-Pattern 2: No Cleanup

```typescript
// BAD - Leaves data pollution
test('Create order', async ({ graph }) => {
    const order = await new SalesOrder(graph).create({ /* ... */ });
    expect(order._id).toBeDefined();
    // No cleanup!
});
```

**Why It's Bad**: Database fills with test data. Unique constraints fail. Performance degrades over time.

### ❌ Anti-Pattern 3: Hard-coded Dependencies

```typescript
// BAD - Relies on database seeding
test('Update customer', async ({ graph }) => {
    const customer = await Customer.get('#CUST001'); // Assumes exists
    await Customer.update('#CUST001', { name: 'Updated' });
});
```

**Why It's Bad**: Test fails in clean environments. Coupling to specific data. Environment-specific failures.

### ❌ Anti-Pattern 4: Test Dependencies (Serial Execution)

```typescript
// BAD - Tests must run in sequence
test.describe.serial('Order Processing', () => {
    test('Step 1: Create order', async () => { /* ... */ });
    test('Step 2: Confirm order', async () => { /* ... */ });
    test('Step 3: Ship order', async () => { /* ... */ });
});
```

**Why It's Bad**: Cannot parallelize. One failure blocks all subsequent tests. Not scalable.

### ❌ Anti-Pattern 5: Missing Tags

```typescript
// BAD - No status tag
test('Create customer', async ({ graph }) => {
    // CI pipeline doesn't know how to categorize this
});
```

**Why It's Bad**: Cannot control test execution in CI. No quarantine strategy. Pipeline configuration fails.

### ❌ Anti-Pattern 6: Polling Synchronous Operations

```typescript
// BAD - Unnecessary polling for CRUD
await expect(async () => {
    const customer = await customerClass.get(customerId);
    expect(customer.name).toBe('Test');
}).toPass({ timeout: 30000 });
```

**Why It's Bad**: Wastes execution time. Masks synchronous failures. False sense of stability.

---

## Example Test Templates

### Template 1: Simple CRUD Test

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { Customer } from '@lib';

test.describe('Customer CRUD Operations', () => {
    let customer: any;

    test.afterEach(async ({ graph }) => {
        if (customer?._id) {
            await Customer.delete(customer._id).catch(() => {});
        }
    });

    test('@release Create customer with valid data', async ({ graph }) => {
        customer = await new Customer(graph).create({
            name: `Customer-${Date.now()}`,
            email: `test-${Date.now()}@example.com`,
        });

        expect(customer._id).toBeDefined();
        expect(customer.name).toContain('Customer-');
    });
});
```

### Template 2: Complex Business Flow

```typescript
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { SalesOrder, Customer, Site, Item } from '@lib';

test.describe('Sales Order Flow', () => {
    const testData: Record<string, any> = {};

    test.beforeEach(async ({ graph }) => {
        testData.customer = await new Customer(graph).create({
            name: `Customer-${Date.now()}`,
        });
        testData.site = await new Site(graph).create({
            name: `Site-${Date.now()}`,
        });
        testData.item = await new Item(graph).create({
            name: `Item-${Date.now()}`,
            price: 100,
        });
    });

    test.afterEach(async ({ graph }) => {
        const cleanup = [
            { id: testData.order?._id, fn: () => SalesOrder.delete(testData.order._id) },
            { id: testData.item?._id, fn: () => Item.delete(testData.item._id) },
            { id: testData.site?._id, fn: () => Site.delete(testData.site._id) },
            { id: testData.customer?._id, fn: () => Customer.delete(testData.customer._id) },
        ];

        for (const resource of cleanup) {
            if (resource.id) {
                await resource.fn().catch(() => {});
            }
        }
    });

    test('@release Create and confirm sales order', async ({ graph }) => {
        testData.order = await new SalesOrder(graph).create({
            customer: testData.customer._id,
            site: testData.site._id,
            items: [{ item: testData.item._id, quantity: 10 }],
        });

        expect(testData.order.status).toBe('Draft');

        await salesOrderClass.confirm(testData.order._id);
        testData.order = await salesOrderClass.get(testData.order._id);

        expect(testData.order.status).toBe('Confirmed');
    });
});
```

---

## Running Tests

### Basic Commands

```bash
# Run all tests
pnpm test

# Run tests without layer loading (faster)
pnpm test:quick

# Run with detailed logging
pnpm test:debug

# Run specific test file
pnpm test tests/sales/order.spec.ts

# Run tests matching pattern
pnpm test --grep "sales order"
```

### Tag-Based Execution

```bash
# Run only release tests
pnpm test --grep "@release"

# Run only development tests
pnpm test --grep "@development"

# Exclude flaky tests (CI default)
pnpm test --grep-invert "@flaky"

# Run only flaky tests (for monitoring)
pnpm test --grep "@flaky"
```

### Debugging

```bash
# Enable GraphQL logging
PLAYWRIGHT_GRAPH_LOG=true pnpm test

# Show server logs
DEBUG=pw:webserver pnpm test

# Skip layer loading
PLAYWRIGHT_SKIP_LAYER_LOAD=true pnpm test

# Combine multiple flags
PLAYWRIGHT_GRAPH_LOG=true DEBUG=pw:webserver pnpm test
```

### Parallel Execution

```bash
# Run with specific number of workers
pnpm test --workers=4

# Run with maximum parallelization
pnpm test --workers=100%

# Run specific shard (for CI)
pnpm test --shard=1/4
```

---

## CI/CD Integration

### Pipeline Configuration

The test suite is designed for horizontal scaling through sharding:

```yaml
# azure-pipelines.yml
strategy:
  matrix:
    shard1: { SHARD: '1/4' }
    shard2: { SHARD: '2/4' }
    shard3: { SHARD: '3/4' }
    shard4: { SHARD: '4/4' }

steps:
  - task: Bash@3
    displayName: 'Run API Tests'
    inputs:
      targetType: 'inline'
      script: |
        # Exclude flaky tests from main pipeline
        pnpm test --grep-invert "@flaky" --shard=$(SHARD)

  - task: PublishPipelineArtifact@1
    inputs:
      targetPath: 'blob-report'
      artifactName: 'blob-report-$(SHARD)'
```

### Sharding Configuration

```typescript
// playwright.config.ts
export default defineConfig({
    fullyParallel: true, // Required for optimal shard distribution
    workers: process.env.CI ? 4 : undefined,
    retries: process.env.CI ? 2 : 0,
    maxFailures: process.env.CI ? 20 : undefined, // Fail fast in CI
    use: {
        trace: 'on-first-retry', // Capture traces only on failures
    },
});
```

### Test Execution Strategy

1. **Sharding**: Tests split across N parallel agents
2. **Full Parallelization**: `fullyParallel: true` ensures test-level distribution
3. **Fail-Fast**: Pipeline aborts after 20 failures (indicates systemic issue)
4. **Retry Logic**: Up to 2 retries in CI for transient failures
5. **Trace Capture**: Only on first retry (performance optimization)

### Report Merging

```bash
# After all shards complete, merge reports
npx playwright merge-reports --reporter html blob-report-*
```

---

## Quick Reference Checklist

Before committing a test, verify:

- [ ] ✅ Test has `@release`, `@development`, or `@flaky` tag in description
- [ ] ✅ Test creates all required data in `beforeEach` or test body
- [ ] ✅ Test has `afterEach` hook for cleanup
- [ ] ✅ Cleanup is defensive (handles missing IDs, catches errors)
- [ ] ✅ No hard-coded entity IDs (no `#CUST001`, `#DE072`, etc.)
- [ ] ✅ Test can run independently (`pnpm test tests/path/to/test.spec.ts`)
- [ ] ✅ Test uses unique identifiers (timestamps, UUIDs) for entity names
- [ ] ✅ Async operations use `toPass()` with explicit timeouts
- [ ] ✅ Test follows naming convention: `@tag Description of what is tested`
- [ ] ✅ Cleanup deletes resources in reverse order (children first)

---

## Performance Guidelines

### Timeout Values

Standardize timeout values across the test suite:

```typescript
const ALLOCATION_TIMEOUT = 60000;  // 60s - Allocation, background jobs
const STANDARD_TIMEOUT = 30000;    // 30s - Most async operations
const QUICK_TIMEOUT = 5000;        // 5s  - Status checks
```

### Optimization Tips

1. **Use `test:quick`** during development (skips layer loading)
2. **Run specific test files** instead of full suite
3. **Limit parallelization** on local machines (`--workers=2`)
4. **Use `--grep`** to run subsets of tests
5. **Provision minimal data** (don't create unused entities)

---

## Appendix: Common Issues and Solutions

### Issue: Test Fails in CI But Passes Locally

**Cause**: Hard-coded dependencies or shared state

**Solution**: 
1. Run test in isolation: `pnpm test tests/path/to/test.spec.ts`
2. Check for hard-coded IDs
3. Verify cleanup is working
4. Add programmatic data provisioning

### Issue: "Email Already Exists" Error

**Cause**: Missing or incomplete cleanup

**Solution**:
1. Add `afterEach` hook
2. Use unique timestamps in email: `` test-${Date.now()}@example.com ``
3. Check cleanup order (delete children first)

### Issue: Test Times Out Waiting for Status

**Cause**: Incorrect polling pattern or missing event

**Solution**:
1. Verify operation is genuinely async
2. Check timeout value is appropriate
3. Reload entity in polling loop (don't cache)
4. Add logging to see what status is returned

### Issue: Cascading Failures After One Test Fails

**Cause**: Shared state or missing isolation

**Solution**:
1. Ensure each test creates own data
2. Verify no global variables
3. Check for `test.describe.serial()` usage
4. Add defensive cleanup

---


