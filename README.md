# API Tests Training & Documentation

Welcome to the Xtrem API Testing framework! This document provides comprehensive guidance for developers and QA engineers to create, run, and maintain API tests using Playwright.

## Table of Contents

1. [Quick Start](#quick-start)
2. [Framework Architecture](#framework-architecture)
3. [Running Tests](#running-tests)
4. [Adding New Tests](#adding-new-tests)
5. [Creating Test Utilities](#creating-test-utilities)
6. [Tagging Strategy](#tagging-strategy)
7. [Example Template](#example-template)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)

---

## Quick Start

### Prerequisites

- Node.js 18+ and pnpm installed
- Project dependencies installed: `pnpm install`
- Application running locally or ability to start it via Playwright

### First Test Run

```bash
# Navigate to api-test directory
cd services/api-test

# Run all tests
pnpm test

# Run tests for a specific module
pnpm test tests/master-data/

# Run tests with specific tags
pnpm test:development
```

---

## Framework Architecture

### Overview

```
services/api-test/
├── lib/                              # Shared utilities and helper classes
│   ├── index.ts                      # Typed test fixture export
│   ├── functions/
│   │   └── random.ts                 # Random data generation utilities
│   ├── master-data/                  # Master data domain helpers
│   │   ├── business-entity.ts
│   │   ├── customer.ts
│   │   ├── item.ts
│   │   ├── site.ts
│   │   ├── supplier.ts
│   │   └── index.ts
│   ├── sales/                        # Sales domain helpers
│   │   ├── sales-order.ts
│   │   └── index.ts
│   ├── system/                       # System domain helpers
│   │   └── index.ts
│   └── reporters/
│       └── error-summary-reporter.ts # Custom error reporting
├── tests/                             # Test files organized by domain
│   ├── master-data/
│   │   ├── customer.spec.ts
│   │   ├── item.spec.ts
│   │   ├── supplier.spec.ts
│   │   ├── business-entity.spec.ts
│   │   └── site.spec.ts
│   ├── sales/
│   │   └── order.spec.ts
│   └── system/
│       └── service-option.spec.ts
├── playwright.config.ts              # Playwright configuration
├── package.json                      # NPM scripts and dependencies
├── tsconfig.json                     # TypeScript configuration
└── setup.ts                          # Global setup (data layer loading)
```

### Key Technologies

- **Playwright**: Modern testing framework with excellent API testing support
- **TypeScript**: Full type safety for GraphQL API interactions
- **@sage/xtrem-playwright**: Framework utilities for Xtrem-specific testing patterns
- **@sage/xtrem-client**: Typed GraphQL client for API interaction
- **BaseNode**: Generic class for CRUD operations on any GraphQL entity

### Architecture Layers

#### 1. **Fixtures Layer** (`services/api-test/lib/index.ts`)

Provides typed GraphQL fixtures with full API type safety:

```typescript
export const test: XtremTestType<GraphApi> = createXtremTest<GraphApi>();
```

**Available Fixtures:**

- `graph`: Read-only GraphQL queries
- `graphMutation`: GraphQL mutations and write operations

#### 2. **Helper Classes Layer** (`lib/master-data/`, `lib/sales/`, etc.)

Domain-specific classes extending `BaseNode` that encapsulate entity operations:

```typescript
export class CustomerClass extends BaseNode<GraphApi, Customer, CustomerInput, CustomerType> {
    get nodeName() { return '@sage/xtrem-master-data/Customer'; }
    get defaultSelector() { return customerSelector; }
    async getRandomData(createData?: CustomerInput): Promise<CustomerInput> { ... }
}
```

**Provides:**

- `query()` - Retrieve entities with filters and selectors
- `create()` - Create new entity with random data
- `update()` - Update entity properties
- `delete()` - Delete entity
- `queryRandom()` - Query with automatic random data generation

#### 3. **Utility Functions Layer** (`lib/functions/`)

Pure functions for generating test data:

```typescript
export function randomString(length: number, charset = 'abcdefghijklmnopqrstuvwxyz'): string;
export function randomInt(options: { min?: number; max: number; not?: number[] }): number;
```

#### 4. **Test Layer** (`tests/`)

Actual test files using the fixtures and helpers:

```typescript
test('should create a customer', async ({ graph }) => {
    const customer = await customerClass.create();
    expect(customer._id).toBeDefined();
});
```

---

## Running Tests

### Available Commands

| Command                 | Purpose                                   | Flags                                             |
| ----------------------- | ----------------------------------------- | ------------------------------------------------- |
| `pnpm test`             | Run all tests with standard configuration | -                                                 |
| `pnpm test:quick`       | Skip layer loading for faster execution   | `PLAYWRIGHT_SKIP_LAYER_LOAD=true`                 |
| `pnpm test:debug`       | Run with GraphQL logging and server logs  | `PLAYWRIGHT_GRAPH_LOG=true`, `DEBUG=pw:webserver` |
| `pnpm test:all`         | Run with chromium project explicitly      | -                                                 |
| `pnpm test:development` | Run only @development tagged tests        | -                                                 |
| `pnpm test:release`     | Run only @release tagged tests            | -                                                 |
| `pnpm test:flaky`       | Run only @flaky tagged tests              | -                                                 |

### Running Tests Locally

#### Step 1: Ensure Dependencies Are Installed

```bash
cd services/api-test
pnpm install
```

#### Step 2: Start Services (if required)

The test framework will automatically start the server via the `serverPath` configuration. For manual startup:

```bash
# From the main directory
pnpm start
```

#### Step 3: Run Tests

```bash
# Basic run
pnpm test

# Run with logging
PLAYWRIGHT_GRAPH_LOG=true pnpm test

# Run single file
pnpm test tests/master-data/customer.spec.ts

# Run with specific tags
pnpm test --grep @development

# Run with watch mode (development)
pnpm test --watch
```

### Running Tests in CI

The CI pipeline automatically:

1. **Shards tests** across multiple workers using `PLAYWRIGHT_SHARD_TOTAL` and `PLAYWRIGHT_SHARD_CURRENT`
2. **Retries failed tests** (1 retry in CI, 0 locally)
3. **Limits workers** to 3 in CI for resource efficiency
4. **Generates reports** in HTML format

CI configuration is defined in `services/api-test/playwright.config.ts`:

```typescript
if (process.env.CI === 'true') {
    const total = Number(process.env.PLAYWRIGHT_SHARD_TOTAL || 1);
    const current = Number(process.env.PLAYWRIGHT_SHARD_CURRENT || 1);
    if (total > 1) {
        shardConfig = { total, current };
    }
}
```

### Environment Variables

| Variable                     | Purpose                                   | Example        |
| ---------------------------- | ----------------------------------------- | -------------- |
| `PLAYWRIGHT_SKIP_LAYER_LOAD` | Skip loading test data layer              | `true`         |
| `PLAYWRIGHT_GRAPH_LOG`       | Enable GraphQL request/response logging   | `true`         |
| `DEBUG`                      | Enable debug output                       | `pw:webserver` |
| `CI`                         | Set CI mode (affects retries and workers) | `true`         |

**Usage:**

```bash
# Multiple variables
PLAYWRIGHT_GRAPH_LOG=true PLAYWRIGHT_SKIP_LAYER_LOAD=true pnpm test

# With debug output
DEBUG=pw:webserver PLAYWRIGHT_GRAPH_LOG=true pnpm test
```

---

## Adding New Tests

### Step 1: Identify Your Domain

Determine if your test belongs to existing domains or a new one:

- **master-data**: Customer, Supplier, Item, Site, BusinessEntity
- **sales**: Orders, quotes, returns
- **system**: System configuration, service options, feature flags

### Step 2: Create the Test File

Create a new `.spec.ts` file in the appropriate domain directory:

```typescript
// tests/master-data/partner.spec.ts
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { MasterData } from '../../lib/master-data';

test.describe('Partner', () => {
    test('@development Partner CRUD operations', async ({ graph }) => {
        const partnerClass = new MasterData.Partner(graph);

        // Your test steps here
    });
});
```

### Step 3: Use the Fixture

Access the typed GraphQL fixture:

```typescript
test('should query partners', async ({ graph }) => {
    const partners = await graph.Partner.query({});
    expect(partners).toBeDefined();
});
```

### Step 4: Organize with Test Steps

Use Playwright's `test.step()` for better reporting:

```typescript
test('should perform partner lifecycle', async ({ graph }) => {
    let partnerId: string;

    await test.step('Create a partner', async () => {
        const result = await graph.Partner.create({
            code: 'PARTNER-001',
            name: 'Test Partner',
        });
        partnerId = result._id;
        expect(partnerId).toBeDefined();
    });

    await test.step('Update partner', async () => {
        const updated = await graph.Partner.update({
            _id: partnerId,
            name: 'Updated Partner',
        });
        expect(updated.name).toBe('Updated Partner');
    });

    await test.step('Delete partner', async () => {
        await graph.Partner.delete({ _id: partnerId });
    });
});
```

### Step 5: Tag Your Tests

Always tag your tests appropriately (see Tagging Strategy):

```typescript
test('@development Partner CRUD operations', async ({ graph }) => { ... });
test('@release Verify partner API stability', async ({ graph }) => { ... });
test('@flaky Partner sync operations', async ({ graph }) => { ... });
```

---

## Creating Test Utilities

### Pattern 1: Domain Helper Class (Recommended)

Extend `BaseNode` for entity-specific operations:

```typescript
// lib/master-data/partner.ts
import { BaseNode } from '@sage/xtrem-playwright';
import { Partner, GraphApi } from '@sage/xtrem-master-data-api';
import { PartnerInput } from '@sage/xtrem-master-data-api-partial';
import { randomString } from '../functions/random';

const partnerSelector = {
    _id: true,
    id: true,
    name: true,
    code: true,
    isActive: true,
} as const;

export type PartnerType = {
    _id: string;
    id: string;
    name: string;
    code: string;
    isActive: boolean;
};

export class PartnerClass extends BaseNode<GraphApi, Partner, PartnerInput, PartnerType> {
    get nodeName() {
        return '@sage/xtrem-master-data/Partner' as keyof GraphApi;
    }

    get defaultSelector() {
        return partnerSelector;
    }

    async getRandomData(createData?: PartnerInput): Promise<PartnerInput> {
        const code = randomString(6).toUpperCase();
        return {
            code,
            name: `Partner ${code}`,
            isActive: true,
            ...createData,
        };
    }

    // Add custom methods for domain-specific operations
    async queryActiveOnly() {
        return this.query({ filter: { isActive: true } });
    }

    async createInactive() {
        return this.create({ isActive: false });
    }
}
```

### Pattern 2: Pure Utility Functions

For common operations across multiple entities:

```typescript
// lib/functions/date-helpers.ts
export function getNextMonday(): Date {
    const today = new Date();
    const nextMonday = new Date(today);
    nextMonday.setDate(today.getDate() + ((1 + 7 - today.getDay()) % 7));
    return nextMonday;
}

export function formatDateForApi(date: Date): string {
    return date.toISOString().split('T')[0];
}
```

### Pattern 3: Setup Fixtures

For common test data setup:

```typescript
// lib/functions/setup-fixtures.ts
import { Graph } from '@sage/xtrem-client';
import { GraphApi } from '@sage/xtrem-master-data-api';
import { MasterData } from '../master-data';

export async function setupTestPartners(graph: Graph<GraphApi>, count: number = 3) {
    const partnerClass = new MasterData.Partner(graph);
    const partners = [];

    for (let i = 0; i < count; i++) {
        partners.push(await partnerClass.create());
    }

    return partners;
}
```

### Pattern 4: Assertion Helpers

For complex validations:

```typescript
// lib/functions/assertions.ts
import { expect } from '@playwright/test';

export function assertPartnerValid(partner: any) {
    expect(partner._id).toBeDefined();
    expect(partner.code).toBeTruthy();
    expect(partner.name).toBeTruthy();
    expect(typeof partner.isActive).toBe('boolean');
}

export function assertPartnerExists(partners: any[], partnerId: string) {
    const found = partners.find(p => p._id === partnerId);
    expect(found).toBeDefined(`Partner ${partnerId} should exist`);
}
```

### Pattern 5: Mock Data Builders

For complex entity creation:

```typescript
// lib/functions/partner-builder.ts
import { PartnerInput } from '@sage/xtrem-master-data-api-partial';
import { randomString } from './random';

export class PartnerBuilder {
    private data: PartnerInput = {};

    withCode(code: string) {
        this.data.code = code;
        return this;
    }

    withName(name: string) {
        this.data.name = name;
        return this;
    }

    withStatus(isActive: boolean) {
        this.data.isActive = isActive;
        return this;
    }

    build(): PartnerInput {
        return {
            code: randomString(6),
            name: randomString(10),
            isActive: true,
            ...this.data,
        };
    }
}

// Usage
const partnerData = new PartnerBuilder().withCode('CUST-001').withStatus(false).build();
```

---

## Tagging Strategy

Tags enable selective test execution and CI/CD pipeline integration. Use meaningful tags to organize your tests:

### Standard Tags

#### `@development`

**Use for:**

- Feature development and integration testing
- Tests that verify core functionality
- Tests using recent/unstable APIs

**Run with:** `pnpm test:development`

```typescript
test('@development Customer CRUD operations', async ({ graph }) => {
    // Tests that verify customer creation, update, deletion
});
```

#### `@release`

**Use for:**

- Stable, production-ready tests
- Tests verifying API contracts
- Tests that should always pass in release builds

**Run with:** `pnpm test:release`

```typescript
test('@release Customer API stability', async ({ graph }) => {
    // Tests critical for product releases
});
```

#### `@flaky`

**Use for:**

- Tests known to be intermittently failing
- Tests with external dependencies
- Tests awaiting fixes

**Run with:** `pnpm test:flaky`

```typescript
test('@flaky Partner sync operations', async ({ graph }) => {
    // Tests that sometimes fail due to timing or external factors
});
```

### Combining Tags

Use pipe syntax to include multiple tags:

```typescript
test('@development|@release', async ({ graph }) => {
    // Runs in both development and release suites
});
```

### Using Tags in CI

CI pipelines can target specific tag suites:

```bash
# In CI pipeline
pnpm test --grep @release  # Only release tests
pnpm test --grep @development  # Only development tests
```

---

## Example Template

Use this template when creating new tests:

```typescript
/**
 * Tests for Partner entity CRUD operations
 * Domain: master-data
 * Coverage: Create, Read, Update, Delete operations
 */
import { expect } from '@playwright/test';
import { test } from '../../lib';
import { MasterData } from '../../lib/master-data';

test.describe('Partner', () => {
    // Basic CRUD test
    test('@development Partner CRUD operations', async ({ graph }) => {
        const partnerClass = new MasterData.Partner(graph);
        let createdPartner: Awaited<ReturnType<typeof partnerClass.create>>;

        // CREATE
        await test.step('Create a partner', async () => {
            createdPartner = await partnerClass.create();
            expect(createdPartner).toBeDefined();
            expect(createdPartner._id).toBeDefined();
            expect(createdPartner.code).toBeTruthy();
        });

        // READ
        await test.step('Query the created partner', async () => {
            const queried = await partnerClass.query({
                filter: { _id: createdPartner._id },
            });
            expect(queried.length).toBe(1);
            expect(queried[0].code).toBe(createdPartner.code);
        });

        // UPDATE
        await test.step('Update partner properties', async () => {
            const updated = await partnerClass.update({
                _id: createdPartner._id,
                name: 'Updated Partner Name',
            });
            expect(updated.name).toBe('Updated Partner Name');
        });

        // DELETE
        await test.step('Delete the partner', async () => {
            await partnerClass.delete({ _id: createdPartner._id });
            const deleted = await partnerClass.query({
                filter: { _id: createdPartner._id },
            });
            expect(deleted.length).toBe(0);
        });
    });

    // Filtered query test
    test('@development Query partners with filters', async ({ graph }) => {
        const partnerClass = new MasterData.Partner(graph);

        await test.step('Create multiple partners', async () => {
            await partnerClass.create();
            await partnerClass.create({ isActive: false });
        });

        await test.step('Query only active partners', async () => {
            const active = await partnerClass.query({
                filter: { isActive: true },
            });
            expect(active.length).toBeGreaterThan(0);
            expect(active.every(p => p.isActive)).toBe(true);
        });
    });

    // Complex scenario test
    test('@release Partner lifecycle with relationships', async ({ graph }) => {
        const partnerClass = new MasterData.Partner(graph);
        const siteClass = new MasterData.Site(graph);

        await test.step('Setup: Create partner and site', async () => {
            // Test setup logic
        });

        await test.step('Associate partner with site', async () => {
            // Association logic
        });

        await test.step('Verify associations persisted', async () => {
            // Verification logic
        });
    });

    // Edge case test
    test('@flaky Partner with special characters', async ({ graph }) => {
        const partnerClass = new MasterData.Partner(graph);

        await test.step('Create partner with special characters', async () => {
            const special = await partnerClass.create({
                name: 'Partner & Co. (Special) [Test]',
            });
            expect(special.name).toContain('&');
        });
    });
});
```

---

## Best Practices

### 1. **Test Independence**

Each test should be completely independent and not rely on other tests:

```typescript
// ✅ GOOD - Each test creates its own data
test('@development Create customer A', async ({ graph }) => {
    const customer = await graph.Customer.create({ name: 'Customer A' });
    expect(customer._id).toBeDefined();
});

test('@development Create customer B', async ({ graph }) => {
    const customer = await graph.Customer.create({ name: 'Customer B' });
    expect(customer._id).toBeDefined();
});

// ❌ BAD - Test B depends on Test A
test('@development Create customer', async ({ graph }) => {
    global.customerId = await graph.Customer.create();
});

test('@development Update customer', async ({ graph }) => {
    // Depends on customerId from previous test!
    await graph.Customer.update({ _id: global.customerId });
});
```

### 2. **Use Test Steps for Organization**

Break tests into logical steps for better reporting and debugging:

```typescript
// ✅ GOOD
test('@development Order workflow', async ({ graph }) => {
    let orderId: string;

    await test.step('Create order', async () => {
        const order = await graph.Order.create();
        orderId = order._id;
    });

    await test.step('Add items to order', async () => {
        await graph.OrderItem.create({ orderId });
    });

    await test.step('Submit order', async () => {
        await graph.Order.submit({ _id: orderId });
    });
});

// ❌ POOR - No step organization
test('@development Order workflow', async ({ graph }) => {
    const order = await graph.Order.create();
    await graph.OrderItem.create({ orderId: order._id });
    await graph.Order.submit({ _id: order._id });
});
```

### 3. **Meaningful Assertions**

Use clear assertions with helpful messages:

```typescript
// ✅ GOOD
test('@development Verify customer creation', async ({ graph }) => {
    const customer = await graph.Customer.create();

    expect(customer._id, 'Customer ID should be generated').toBeDefined();
    expect(customer.code, 'Customer code should be set').toBeTruthy();
    expect(customer.isActive, 'Customer should be active by default').toBe(true);
});

// ❌ POOR - Generic assertions
test('@development Create customer', async ({ graph }) => {
    const customer = await graph.Customer.create();
    expect(customer).toBeDefined();
    expect(customer._id).toBeDefined();
});
```

### 4. **Leverage Helper Classes**

Use domain helper classes instead of raw graph queries:

```typescript
// ✅ GOOD
test('@development Customer operations', async ({ graph }) => {
    const customerClass = new MasterData.Customer(graph);
    const customer = await customerClass.create();
    expect(customer._id).toBeDefined();
});

// ❌ POOR - Direct graph manipulation
test('@development Customer operations', async ({ graph }) => {
    const customer = await graph.Customer.query({});
    // Manually managing selectors and filters
});
```

### 5. **Clean Up Test Data**

Always delete created test data, especially for @release tests:

```typescript
// ✅ GOOD
test('@release Customer operations', async ({ graph }) => {
    const customerClass = new MasterData.Customer(graph);
    const customer = await customerClass.create();

    try {
        expect(customer._id).toBeDefined();
    } finally {
        // Cleanup happens even if test fails
        await customerClass.delete({ _id: customer._id });
    }
});

// ❌ POOR - No cleanup
test('@release Customer operations', async ({ graph }) => {
    const customer = await graph.Customer.create();
    expect(customer._id).toBeDefined();
    // Test data left behind!
});
```

### 6. **Use Appropriate Test Tags**

Tag tests according to their stability and lifecycle:

```typescript
// ✅ GOOD - Clear intent
test('@development Experimental feature', async ({ graph }) => { ... });
test('@release Core API contract', async ({ graph }) => { ... });
test('@flaky Sync operations with timeout', async ({ graph }) => { ... });

// ❌ POOR - Unclear tags
test('Feature test', async ({ graph }) => { ... });
test('Verify API', async ({ graph }) => { ... });
```

### 7. **Handle Random Data Properly**

Store generated IDs for later queries:

```typescript
// ✅ GOOD
test('@development Query created customer', async ({ graph }) => {
    const customerClass = new MasterData.Customer(graph);
    const created = await customerClass.create();
    const customerId = created._id;

    const queried = await customerClass.query({
        filter: { _id: customerId },
    });
    expect(queried[0]._id).toBe(customerId);
});

// ❌ POOR - Assume specific data exists
test('@development Query customer', async ({ graph }) => {
    const customers = await graph.Customer.query({});
    expect(customers.length).toBeGreaterThan(0);
});
```

### 8. **Organize Tests by Domain**

Keep related tests together and organized:

```
tests/
├── master-data/         # Master data entities
│   ├── customer.spec.ts
│   ├── supplier.spec.ts
│   └── item.spec.ts
├── sales/              # Sales transactions
│   ├── order.spec.ts
│   └── quote.spec.ts
└── system/             # System configuration
    └── service-option.spec.ts
```

---

## Troubleshooting

### Issue: Tests Fail with "Cannot query field"

**Cause**: GraphQL selector doesn't match the API schema.

**Solution**:

1. Verify the field exists in the GraphQL API definition
2. Check TypeScript types for correct field names
3. Update the selector in your helper class

```typescript
// ✅ CORRECT
const selector = {
    _id: true,
    code: true,
    isActive: true,
};

// ❌ WRONG - Field doesn't exist
const selector = {
    _id: true,
    customField: true, // This field doesn't exist!
};
```

### Issue: "No \_id to update" Error

**Cause**: Attempting to update without an entity ID.

**Solution**: Ensure entity is created or ID is provided:

```typescript
// ✅ CORRECT
const customer = await customerClass.create();
await customerClass.update({ name: 'New Name' }); // Uses created entity

// ❌ WRONG - No entity created
const customerClass = new MasterData.Customer(graph);
await customerClass.update({ name: 'New Name' }); // No ID!
```

### Issue: Layer Load Takes Too Long

**Cause**: Test data layer loading from database.

**Solution**: Use quick mode for development:

```bash
PLAYWRIGHT_SKIP_LAYER_LOAD=true pnpm test
```

**Note**: Only skip layer load if test data already exists.

### Issue: Tests Pass Locally but Fail in CI

**Cause**: Timing issues, parallel test conflicts, or environment differences.

**Solutions**:

1. **Increase timeouts** for slow operations:

    ```typescript
    test.setTimeout(60000); // 60 seconds
    ```

2. **Mark as flaky** if intermittent:

    ```typescript
    test('@flaky Slow operation', async ({ graph }) => { ... });
    ```

3. **Check environment variables** in CI:
    ```bash
    PLAYWRIGHT_SKIP_LAYER_LOAD=true pnpm test  # Local test
    # vs CI configuration
    ```

### Issue: GraphQL Query Returns Empty

**Cause**: Querying with incorrect filter or data doesn't exist.

**Solution**:

1. Enable GraphQL logging:

    ```bash
    PLAYWRIGHT_GRAPH_LOG=true pnpm test
    ```

2. Check filters match your created data:
    ```typescript
    const customer = await customerClass.create({ code: 'CUST-001' });
    const found = await customerClass.query({
        filter: { code: 'CUST-001' }, // Match what you created
    });
    ```

### Issue: Random Data Conflicts

**Cause**: Random string generation creates duplicates.

**Solution**: Extend random string length or use timestamps:

```typescript
// In lib/functions/random.ts
export function randomString(length: number = 12, charset = 'abcdefghijklmnopqrstuvwxyz0123456789') {
    let res = '';
    while (length) {
        length--;
        res += charset[(unsafeRandom() * charset.length) | 0];
    }
    return res;
}

// Or use timestamps
export function uniqueCode(): string {
    return `CODE-${Date.now()}-${randomInt({ max: 1000 })}`;
}
```

---

## Additional Resources

### Internal Documentation

- **Playwright Documentation**: https://playwright.dev/docs/api-testing
- **@sage/xtrem-playwright**: Framework utilities and fixtures
- **@sage/xtrem-client**: GraphQL client and type definitions
- **BaseNode Implementation**: Located in `platform/testing/xtrem-playwright/lib/base-node.ts`

### Running Tests Quick Reference

| Scenario               | Command                                                  |
| ---------------------- | -------------------------------------------------------- |
| All tests              | `pnpm test`                                              |
| Development tests only | `pnpm test:development`                                  |
| Release tests only     | `pnpm test:release`                                      |
| Specific test file     | `pnpm test tests/master-data/customer.spec.ts`           |
| Watch mode             | `pnpm test --watch`                                      |
| With GraphQL logs      | `PLAYWRIGHT_GRAPH_LOG=true pnpm test`                    |
| Skip layer load        | `PLAYWRIGHT_SKIP_LAYER_LOAD=true pnpm test`              |
| With debugging         | `DEBUG=pw:webserver PLAYWRIGHT_GRAPH_LOG=true pnpm test` |

### File Structure Quick Reference

- **Write tests**: `tests/[domain]/[feature].spec.ts`
- **Create helpers**: `lib/[domain]/[entity].ts`
- **Pure functions**: `lib/functions/[utility].ts`
- **Share fixtures**: Update `services/api-test/lib/index.ts`

---

## Getting Help

1. **Check existing tests** - Review `services/api-test/tests/master-data/customer.spec.ts` for examples
2. **Review helpers** - See how `BaseNode` is used in `lib/master-data/*.ts`
3. **Debug with logs** - Use `PLAYWRIGHT_GRAPH_LOG=true` environment variable
4. **Consult framework docs** - See `@sage/xtrem-playwright` documentation
5. **Ask the team** - Schedule knowledge-sharing sessions on API testing patterns
