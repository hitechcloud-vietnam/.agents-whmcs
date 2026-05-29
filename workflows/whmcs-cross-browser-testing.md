# WHMCS Cross-Browser Testing Workflow

## Overview
This workflow guides you through comprehensive cross-browser testing for WHMCS to ensure consistent user experience across all browsers and devices.

## Prerequisites
- WHMCS installation (v8.0+)
- Browser testing tools (BrowserStack, LambdaTest, Playwright)
- Multiple browser versions
- Virtual machines for testing

## Step-by-Step Guide

### Step 1: Define Browser Support Matrix

#### Supported Browsers
| Browser | Minimum Version | Recommended |
|---------|-----------------|-------------|
| Chrome | 90+ | Latest |
| Firefox | 88+ | Latest |
| Safari | 14+ | Latest |
| Edge | 90+ | Latest |
| Opera | 76+ | Latest |
| IE 11 | Deprecated | Not supported |

### Step 2: Set Up Playwright Cross-Browser Testing

#### Configuration
```javascript
// playwright-cross-browser.config.js
const { defineConfig, devices } = require('@playwright/test');

module.exports = defineConfig({
  testDir: './cross-browser-tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: 2,
  workers: process.env.CI ? 1 : 3,
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results/results.json' }],
  ],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    // Desktop browsers
    {
      name: 'Chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'Firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'Safari',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'Edge',
      use: { ...devices['Desktop Edge'] },
    },
    // Mobile browsers
    {
      name: 'iPhone Safari',
      use: { ...devices['iPhone 12'] },
    },
    {
      name: 'iPad Safari',
      use: { ...devices['iPad (gen 7)'] },
    },
    {
      name: 'Android Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Android Samsung',
      use: { ...devices['Samsung Galaxy S21'] },
    },
  ],
});
```

### Step 3: Create Cross-Browser Test Suite

#### Layout Tests
```javascript
// cross-browser-tests/layout.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Cross-Browser Layout Tests', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });

  test('header displays correctly across browsers', async ({ page }) => {
    const header = page.locator('header');
    await expect(header).toBeVisible();
    await expect(header).toContainText('Logo');
    
    // Check menu items
    const menuItems = page.locator('nav a');
    const count = await menuItems.count();
    expect(count).toBeGreaterThan(0);
  });

  test('footer displays correctly', async ({ page }) => {
    const footer = page.locator('footer');
    await expect(footer).toBeVisible();
    await expect(footer).toContainText('202');
  });

  test('responsive layout at desktop size', async ({ page }) => {
    await page.setViewportSize({ width: 1920, height: 1080 });
    
    const sidebar = page.locator('.sidebar');
    const mainContent = page.locator('.main-content');
    
    await expect(sidebar).toBeVisible();
    await expect(mainContent).toBeVisible();
    
    // Sidebar should be on the left
    const sidebarBox = await sidebar.boundingBox();
    const contentBox = await mainContent.boundingBox();
    
    expect(sidebarBox.x).toBeLessThan(contentBox.x);
  });

  test('navigation menu works', async ({ page }) => {
    await page.click('nav a[href="/cart.php"]');
    await expect(page).toHaveURL(/cart\.php/);
  });
});
```

#### Form Tests
```javascript
// cross-browser-tests/forms.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Form Functionality Cross-Browser', () => {
  test('login form works across browsers', async ({ page }) => {
    await page.goto('/clientarea.php');
    
    // Fill login form
    await page.fill('input[name="username"]', 'testuser');
    await page.fill('input[name="password"]', 'testpass');
    
    // Submit
    await page.click('button[type="submit"]');
    
    // Check for errors or success
    const hasError = await page.locator('.error-message').isVisible();
    const hasSuccess = await page.locator('.success-message').isVisible();
    
    // Either should be true (form submitted)
    expect(hasError || hasSuccess).toBeTruthy();
  });

  test('registration form validation', async ({ page }) => {
    await page.goto('/register.php');
    
    // Try to submit empty form
    await page.click('button[type="submit"]');
    
    // Check validation messages appear
    const validationMessages = page.locator('.field-error, .help-block');
    expect(await validationMessages.count()).toBeGreaterThan(0);
  });

  test('dropdown selects work', async ({ page }) => {
    await page.goto('/cart.php');
    
    // Find and interact with dropdown
    const dropdown = page.locator('select[name="billingcycle"]');
    await expect(dropdown).toBeVisible();
    
    await dropdown.selectOption({ index: 1 });
    const selectedValue = await dropdown.inputValue();
    expect(selectedValue).toBeTruthy();
  });

  test('file upload functionality', async ({ page }) => {
    await page.goto('/submitticket.php');
    
    // Check file input exists
    const fileInput = page.locator('input[type="file"]');
    await expect(fileInput).toBeAttached();
  });

  test('date picker works', async ({ page }) => {
    await page.goto('/admin/clients.php?userid=1');
    
    const dateInput = page.locator('input.date-picker, input[type="date"]');
    if (await dateInput.count() > 0) {
      await dateInput.first().fill('2024-12-25');
      const value = await dateInput.first().inputValue();
      expect(value).toBeTruthy();
    }
  });
});
```

#### JavaScript Functionality Tests
```javascript
// cross-browser-tests/javascript.spec.js
const { test, expect } = require('@playwright/test');

test.describe('JavaScript Functionality', () => {
  test('JavaScript alerts work', async ({ page }) => {
    page.on('dialog', async dialog => {
      expect(dialog.type()).toBe('alert');
      expect(dialog.message()).toContain('Test');
      await dialog.accept();
    });
    
    await page.goto('/test-alert.php');
    await page.click('#trigger-alert');
  });

  test('AJAX operations work', async ({ page }) => {
    await page.goto('/cart.php');
    
    // Wait for AJAX to complete
    await page.click('.add-to-cart-btn');
    
    await expect(page.locator('.cart-updated')).toBeVisible({ timeout: 5000 });
  });

  test('modal dialogs work', async ({ page }) => {
    await page.goto('/clientarea.php');
    
    // Open modal
    await page.click('[data-modal="open"]');
    await expect(page.locator('.modal')).toBeVisible();
    
    // Close modal
    await page.click('.modal .close');
    await expect(page.locator('.modal')).not.toBeVisible();
  });

  test('tooltips display correctly', async ({ page }) => {
    await page.goto('/cart.php');
    
    const tooltipTrigger = page.locator('[data-tooltip]').first();
    if (await tooltipTrigger.count() > 0) {
      await tooltipTrigger.hover();
      await expect(page.locator('.tooltip, [role="tooltip"]')).toBeVisible();
    }
  });

  test('drag and drop works', async ({ page }) => {
    await page.goto('/test-drag.php');
    
    const source = page.locator('.draggable');
    const target = page.locator('.droppable');
    
    if (await source.count() > 0 && await target.count() > 0) {
      await source.dragTo(target);
      await expect(target).toHaveClass(/dropped/);
    }
  });
});
```

#### Shopping Cart Tests
```javascript
// cross-browser-tests/cart.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Shopping Cart Cross-Browser', () => {
  test('add item to cart', async ({ page }) => {
    await page.goto('/cart.php');
    
    const addButton = page.locator('.product-item .add-btn').first();
    await addButton.click();
    
    await page.waitForTimeout(500);
    
    // Check cart updated
    const cartCount = page.locator('.cart-count, .cart-badge');
    await expect(cartCount).toBeVisible();
  });

  test('remove item from cart', async ({ page }) => {
    await page.goto('/cart.php');
    
    const removeButton = page.locator('.remove-item').first();
    await removeButton.click();
    
    await page.waitForTimeout(500);
  });

  test('update quantity in cart', async ({ page }) => {
    await page.goto('/cart.php');
    
    const quantityInput = page.locator('.cart-item input[type="number"]').first();
    await quantityInput.fill('3');
    await quantityInput.press('Enter');
    
    await page.waitForTimeout(500);
  });

  test('apply coupon code', async ({ page }) => {
    await page.goto('/cart.php');
    
    await page.fill('#coupon-code', 'SAVE20');
    await page.click('#apply-coupon');
    
    await page.waitForTimeout(1000);
  });

  test('checkout process', async ({ page }) => {
    await page.goto('/cart.php');
    
    await page.click('#checkout-btn');
    await expect(page).toHaveURL(/checkout/);
  });
});
```

### Step 4: Run Cross-Browser Tests

```bash
# Run all browsers
npx playwright test

# Run specific browser
npx playwright test --project=Chromium
npx playwright test --project=Firefox
npx playwright test --project=Safari

# Run mobile only
npx playwright test --project="iPhone Safari"
npx playwright test --project="Android Chrome"

# Run with video recording
npx playwright test --video=on

# Generate report
npx playwright show-report
```

### Step 5: BrowserStack Integration
```javascript
// playwright.browserstack.config.js
const { defineConfig, devices } = require('@playwright/test');

module.exports = defineConfig({
  testDir: './cross-browser-tests',
  reporter: [
    ['html'],
    ['browserstack'],
  ],
  use: {
    browserName: 'chrome',
  },
  projects: [
    {
      name: 'BrowserStack Chrome',
      use: {
        ...devices['Desktop Chrome'],
        browserStackOptions: {
          os: 'Windows',
          osVersion: '11',
          sessionName: 'WHMCS Test',
          buildName: 'WHMC-CI-Build',
        },
      },
    },
    {
      name: 'BrowserStack Safari',
      use: {
        ...devices['Safari'],
        browserStackOptions: {
          os: 'OS X',
          osVersion: 'Monterey',
          sessionName: 'WHMCS Safari Test',
        },
      },
    },
  ],
});
```

### Step 6: Generate Cross-Browser Report
```bash
# Run tests and generate report
npx playwright test --reporter=html,json

# View HTML report
npx playwright show-report

# Check specific browser results
cat test-results/results.json | jq '.suites[].specs[] | select(.projectName=="Safari")'
```

## Common Cross-Browser Issues

| Issue | Affected Browsers | Solution |
|-------|-------------------|----------|
| Flexbox gaps | IE11 | Use alternative layout |
| CSS Grid | IE11 | Use flexbox fallback |
| JavaScript ES6 | Older browsers | Transpile with Babel |
| CSS custom properties | IE11 | Use preprocessor variables |
| Form validation | Safari | Test native validation |
| Date inputs | Safari, IE | Use date picker library |
| File input styling | All | Use custom file input |
| Position: sticky | Older Edge | Use JavaScript fallback |

## Browser-Specific Testing Checklist

### Chrome
- [ ] DevTools console errors
- [ ] Extension compatibility
- [ ] Hardware acceleration

### Firefox
- [ ] CSS grid layout
- [ ] Mixed content handling
- [ ] Privacy settings

### Safari
- [ ] File input behavior
- [ ] Date picker styling
- [ ] WebDriver compatibility

### Edge
- [ ] IE compatibility mode
- [ ] WebDriver issues
- [ ] Edge-specific features

### Mobile
- [ ] Touch events
- [ ] Viewport meta tag
- [ ] Virtual keyboard
- [ ] Device rotation
