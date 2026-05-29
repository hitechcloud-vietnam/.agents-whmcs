# WHMCS End-to-End Testing Workflow

## Overview
This workflow provides a comprehensive guide for creating and executing end-to-end (E2E) tests for WHMCS, simulating real user interactions from browser to database.

## Prerequisites
- WHMCS installation (v8.0+)
- Playwright or Cypress installed
- Selenium WebDriver (alternative)
- Test WHMCS instance with sample data
- Browser drivers installed

## Step-by-Step Guide

### Step 1: Set Up E2E Testing Framework

#### Install Playwright
```bash
npm init -y
npm install -D @playwright/test
npx playwright install chromium
```

#### Configuration
```javascript
// playwright.config.js
const { defineConfig, devices } = require('@playwright/test');

module.exports = defineConfig({
  testDir: './e2e-tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],
  webServer: {
    command: 'php -S localhost:80',
    port: 80,
    reuseExistingServer: !process.env.CI,
  },
});
```

### Step 2: Create Test Helpers
```javascript
// e2e-tests/helpers/whmcs-helper.js
class WHMCSHelper {
  constructor(page) {
    this.page = page;
    this.baseURL = process.env.WHMCS_URL || 'http://localhost';
  }

  async login(username, password) {
    await this.page.goto(`${this.baseURL}/admin/login.php`);
    await this.page.fill('input[name="username"]', username);
    await this.page.fill('input[name="password"]', password);
    await this.page.click('button[type="submit"]');
    await this.page.waitForURL('**/admin/index.php');
  }

  async logout() {
    await this.page.goto(`${this.baseURL}/admin/logout.php`);
  }

  async createClient(clientData) {
    await this.page.goto(`${this.baseURL}/admin/clientsadd.php`);
    await this.page.fill('#firstname', clientData.firstName);
    await this.page.fill('#lastname', clientData.lastName);
    await this.page.fill('#email', clientData.email);
    await this.page.fill('#password2', clientData.password);
    await this.page.click('button[type="submit"]');
    await this.page.waitForSelector('.alert-success');
  }

  async addProductToCart(productId, configOptions = {}) {
    await this.page.goto(`${this.baseURL}/cart.php`);
    await this.page.click(`[data-product-id="${productId}"]`);
    
    for (const [key, value] of Object.entries(configOptions)) {
      await this.page.selectOption(`#configoption${key}`, value);
    }
    
    await this.page.click('button[data-action="addToCart"]');
  }

  async checkout(paymentMethod = 'paypal') {
    await this.page.goto(`${this.baseURL}/cart.php?a=checkout`);
    await this.page.selectOption('#paymentmethod', paymentMethod);
    await this.page.click('#btnCompleteOrder');
    await this.page.waitForSelector('.checkout-success');
  }
}

module.exports = { WHMCSHelper };
```

### Step 3: Write E2E Tests

#### Admin Panel Tests
```javascript
// e2e-tests/admin/client-management.spec.js
const { test, expect } = require('@playwright/test');
const { WHMCSHelper } = require('../helpers/whmcs-helper');

test.describe('Client Management', () => {
  let helper;

  test.beforeEach(async ({ page }) => {
    helper = new WHMCSHelper(page);
    await helper.login('admin', process.env.ADMIN_PASSWORD);
  });

  test.afterEach(async () => {
    await helper.logout();
  });

  test('create new client', async ({ page }) => {
    const clientData = {
      firstName: 'John',
      lastName: 'Doe',
      email: `john.doe.${Date.now()}@example.com`,
      password: 'SecurePass123!',
    };

    await helper.createClient(clientData);

    await expect(page.locator('.alert-success')).toContainText('Client added successfully');
    
    const clientsTable = page.locator('#clients-list');
    await expect(clientsTable).toContainText(clientData.lastName);
  });

  test('edit client details', async ({ page }) => {
    await page.goto(`${process.env.WHMCS_URL}/admin/clients.php?userid=1`);
    
    await page.click('a[href*="action=edit"]');
    await page.fill('#phonenumber', '555-1234');
    await page.click('button[type="submit"]');
    
    await expect(page.locator('.alert-success')).toContainText('Client updated');
  });

  test('delete client', async ({ page }) => {
    await page.goto(`${process.env.WHMCS_URL}/admin/clients.php?userid=1`);
    
    await page.click('a[data-action="delete"]');
    await page.click('button.confirm-delete');
    
    await expect(page.locator('.alert-success')).toContainText('Client deleted');
  });

  test('search clients', async ({ page }) => {
    await page.goto(`${process.env.WHMCS_URL}/admin/clients.php`);
    
    await page.fill('#searchinput', 'john.doe');
    await page.click('button[type="submit"]');
    
    const results = page.locator('.client-result');
    await expect(results.first()).toContainText('john.doe');
  });
});
```

#### Order and Checkout Tests
```javascript
// e2e-tests/orders/order-flow.spec.js
const { test, expect } = require('@playwright/test');
const { WHMCSHelper } = require('../helpers/whmcs-helper');

test.describe('Order and Checkout Flow', () => {
  let helper;

  test.beforeEach(async ({ page }) => {
    helper = new WHMCSHelper(page);
  });

  test('complete order with PayPal', async ({ page }) => {
    // Add product to cart
    await helper.addProductToCart(1, { 1: 'option1', 2: '5' });
    
    // Configure product
    await page.click('button[data-action="configureProduct"]');
    await page.waitForSelector('#configureProductModal');
    await page.click('button[data-action="addToCart"]');
    
    // Go to checkout
    await helper.checkout('paypal');
    
    // Verify order
    await expect(page.locator('.order-number')).toBeVisible();
    await expect(page.locator('.order-status')).toContainText('Pending');
  });

  test('apply coupon code', async ({ page }) => {
    await page.goto(`${process.env.WHMCS_URL}/cart.php`);
    
    await page.fill('#inputCouponCode', 'SAVE20');
    await page.click('button[data-action="applyCoupon"]');
    
    await expect(page.locator('.coupon-success')).toContainText('Coupon applied');
    await expect(page.locator('.order-total')).toContainText('Discount');
  });

  test('remove item from cart', async ({ page }) => {
    await helper.addProductToCart(1);
    
    await page.goto(`${process.env.WHMCS_URL}/cart.php`);
    await page.click('button[data-action="removeItem"]');
    
    await expect(page.locator('.cart-empty')).toContainText('Your cart is empty');
  });
});
```

#### Invoice Tests
```javascript
// e2e-tests/billing/invoice.spec.js
const { test, expect } = require('@playwright/test');
const { WHMCSHelper } = require('../helpers/whmcs-helper');

test.describe('Invoice Management', () => {
  let helper;

  test.beforeEach(async ({ page }) => {
    helper = new WHMCSHelper(page);
    await helper.login('admin', process.env.ADMIN_PASSWORD);
  });

  test('create manual invoice', async ({ page }) => {
    await page.goto(`${process.env.WHMCS_URL}/admin/invoices.php?action=create`);
    
    await page.selectOption('#clientid', '1');
    await page.fill('#itemdescription0', 'Custom Service');
    await page.fill('#itemamount0', '99.99');
    await page.click('button[type="submit"]');
    
    await expect(page.locator('.alert-success')).toContainText('Invoice created');
  });

  test('send invoice email', async ({ page }) => {
    await page.goto(`${process.env.WHMCS_URL}/admin/invoices.php?id=1`);
    
    await page.click('button[data-action="sendInvoice"]');
    
    await expect(page.locator('.alert-success')).toContainText('Invoice sent');
  });

  test('mark invoice as paid', async ({ page }) => {
    await page.goto(`${process.env.WHMCS_URL}/admin/invoices.php?id=1`);
    
    await page.click('button[data-action="markPaid"]');
    await page.waitForSelector('#status', { state: 'visible' });
    
    const status = await page.locator('#status').textContent();
    expect(status).toContain('Paid');
  });
});
```

### Step 4: Run E2E Tests
```bash
# Run all E2E tests
npx playwright test

# Run specific test file
npx playwright test e2e-tests/admin/client-management.spec.js

# Run with UI
npx playwright test --ui

# Run with headed browser
npx playwright test --headed

# Run specific browser
npx playwright test --project=chromium

# Generate report
npx playwright show-report
```

### Step 5: CI/CD Integration
```yaml
# .github/workflows/e2e-tests.yml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  e2e-tests:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          extensions: pdo, pdo_mysql

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: |
          npm ci
          npx playwright install --with-deps chromium

      - name: Setup WHMCS test environment
        run: |
          cp .env.example .env
          php artisan key:generate
          php artisan migrate --seed

      - name: Run E2E tests
        env:
          WHMCS_URL: http://localhost
          ADMIN_PASSWORD: ${{ secrets.ADMIN_PASSWORD }}
        run: npx playwright test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

## Best Practices
- Use page objects for reusable element selectors
- Implement proper waits instead of fixed delays
- Run tests in isolation with unique data
- Take screenshots on failure for debugging
- Use data-testid attributes for stable selectors
- Implement test retry for flaky tests

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Element not found | Check for dynamic IDs, use more specific selectors |
| Test timeout | Increase timeout or check for blocking operations |
| Authentication errors | Verify session handling and cookies |
| Database state issues | Use transactions or cleanup hooks |
| Race conditions | Add explicit waits for async operations |

## Expected Output
```
Running 45 tests using 4 workers

  Admin Client Management
    ✓ create new client (3.2s)
    ✓ edit client details (2.1s)
    ✓ delete client (1.8s)
    ✓ search clients (1.5s)

  Order and Checkout Flow
    ✓ complete order with PayPal (8.5s)
    ✓ apply coupon code (2.3s)
    ✓ remove item from cart (1.2s)

  Invoice Management
    ✓ create manual invoice (3.4s)
    ✓ send invoice email (2.0s)
    ✓ mark invoice as paid (1.9s)

45 passed (45s)
```
