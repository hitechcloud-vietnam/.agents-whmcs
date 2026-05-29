# WHMCS Mobile Testing Workflow

## Overview
This workflow provides comprehensive guidance for testing WHMCS on mobile devices, ensuring responsive design and mobile-specific functionality.

## Prerequisites
- WHMCS installation (v8.0+)
- Mobile testing tools (Playwright, BrowserStack, Xcode Simulator)
- Android Studio for Android testing
- Various mobile devices (optional)

## Step-by-Step Guide

### Step 1: Set Up Mobile Testing Environment

#### Configure Playwright for Mobile
```javascript
// playwright-mobile.config.js
const { defineConfig, devices } = require('@playwright/test');

module.exports = defineConfig({
  testDir: './mobile-tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  reporter: [
    ['html'],
    ['list'],
  ],
  use: {
    baseURL: 'http://localhost',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    // iOS devices
    {
      name: 'iPhone 14 Pro',
      use: { ...devices['iPhone 14 Pro'] },
    },
    {
      name: 'iPhone SE',
      use: { ...devices['iPhone SE'] },
    },
    {
      name: 'iPad Pro',
      use: { ...devices['iPad Pro 11-inch'] },
    },
    // Android devices
    {
      name: 'Pixel 7',
      use: { ...devices['Pixel 7'] },
    },
    {
      name: 'Samsung Galaxy S23',
      use: { ...devices['Samsung Galaxy S21'] },
    },
    {
      name: 'Xiaomi Redmi',
      use: {
        viewport: { width: 393, height: 851 },
        userAgent: 'Mozilla/5.0 (Linux; Android 12; Redmi Note 11) AppleWebKit/537.36',
        deviceScaleFactor: 2.5,
        isMobile: true,
        hasTouch: true,
      },
    },
  ],
});
```

### Step 2: Create Mobile Test Suite

#### Responsive Design Tests
```javascript
// mobile-tests/responsive.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Mobile Responsive Design', () => {
  const viewports = [
    { name: 'iPhone SE', width: 375, height: 667 },
    { name: 'iPhone 14', width: 390, height: 844 },
    { name: 'Pixel 7', width: 412, height: 915 },
    { name: 'iPad Mini', width: 768, height: 1024 },
  ];

  for (const viewport of viewports) {
    test(`layout adapts correctly on ${viewport.name}`, async ({ page }) => {
      await page.setViewportSize({ width: viewport.width, height: viewport.height });
      await page.goto('/');
      
      // Check header adapts
      const header = page.locator('header');
      await expect(header).toBeVisible();
      
      // Check main content is accessible
      const mainContent = page.locator('main, .content, .container');
      await expect(mainContent.first()).toBeVisible();
    });
  }

  test('navigation menu collapses on mobile', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
    await page.goto('/');
    
    // Find hamburger menu or mobile menu
    const mobileMenuButton = page.locator('.navbar-toggler, .mobile-menu-toggle, #mobile-menu');
    
    if (await mobileMenuButton.isVisible()) {
      await mobileMenuButton.click();
      await page.waitForTimeout(300);
      
      // Menu should be visible after click
      const mobileMenu = page.locator('.mobile-nav, .navbar-collapse, .nav-menu');
      await expect(mobileMenu).toBeVisible({ timeout: 2000 });
    }
  });

  test('touch targets are appropriately sized', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
    await page.goto('/');
    
    // Find clickable elements
    const buttons = page.locator('button, a, input[type="submit"]');
    const count = await buttons.count();
    
    for (let i = 0; i < Math.min(count, 10); i++) {
      const button = buttons.nth(i);
      const box = await button.boundingBox();
      
      if (box) {
        // Touch targets should be at least 44x44px
        expect(box.width).toBeGreaterThanOrEqual(24);
        expect(box.height).toBeGreaterThanOrEqual(24);
      }
    }
  });
});
```

#### Mobile Navigation Tests
```javascript
// mobile-tests/navigation.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Mobile Navigation', () => {
  test.beforeEach(async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
    await page.goto('/');
  });

  test('mobile header is visible', async ({ page }) => {
    const header = page.locator('header, .mobile-header');
    await expect(header).toBeVisible();
  });

  test('logo links to homepage', async ({ page }) => {
    const logo = page.locator('header .logo, header img').first();
    if (await logo.isVisible()) {
      await logo.click();
      await expect(page).toHaveURL(/\/$|\/index\.php/);
    }
  });

  test('menu opens and closes correctly', async ({ page }) => {
    // Open menu
    const menuToggle = page.locator('.menu-toggle, .hamburger, [aria-label="Menu"]').first();
    
    if (await menuToggle.isVisible()) {
      await menuToggle.click();
      await page.waitForTimeout(300);
      
      const menu = page.locator('.nav-menu, .mobile-menu, nav ul');
      await expect(menu).toBeVisible();
      
      // Close menu
      const closeButton = page.locator('.menu-close, .close-menu').first();
      
      if (await closeButton.isVisible()) {
        await closeButton.click();
        await page.waitForTimeout(300);
      } else {
        // Try clicking outside menu
        await page.locator('main').click();
        await page.waitForTimeout(300);
      }
    }
  });

  test('footer navigation works', async ({ page }) => {
    await page.goto('/');
    
    // Scroll to footer
    await page.locator('footer').scrollIntoViewIfNeeded();
    
    // Check footer links
    const footerLinks = page.locator('footer a');
    const count = await footerLinks.count();
    expect(count).toBeGreaterThan(0);
  });

  test('breadcrumb navigation works', async ({ page }) => {
    await page.goto('/clientarea.php');
    
    const breadcrumbs = page.locator('.breadcrumbs, .breadcrumb, nav[aria-label="Breadcrumb"]');
    
    if (await breadcrumbs.isVisible()) {
      const links = breadcrumbs.locator('a');
      const count = await links.count();
      
      if (count > 0) {
        await links.first().click();
        await expect(page).toHaveURL(/\//);
      }
    }
  });
});
```

#### Mobile Forms Tests
```javascript
// mobile-tests/forms.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Mobile Forms', () => {
  test.beforeEach(async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
  });

  test('login form on mobile', async ({ page }) => {
    await page.goto('/clientarea.php');
    
    const usernameInput = page.locator('input[name="username"], input[type="email"]');
    const passwordInput = page.locator('input[name="password"]');
    
    await expect(usernameInput).toBeVisible();
    await expect(passwordInput).toBeVisible();
    
    // Fill and submit
    await usernameInput.fill('test@example.com');
    await passwordInput.fill('password123');
    
    await page.locator('button[type="submit"]').click();
    await page.waitForTimeout(1000);
  });

  test('registration form scrolls correctly', async ({ page }) => {
    await page.goto('/register.php');
    
    // Fill first field
    await page.fill('#firstname', 'John');
    
    // Scroll to see if next fields are visible
    await page.locator('#email').scrollIntoViewIfNeeded();
    await page.fill('#email', 'john@example.com');
  });

  test('select dropdowns work on mobile', async ({ page }) => {
    await page.goto('/cart.php');
    
    const select = page.locator('select').first();
    
    if (await select.isVisible()) {
      await select.click();
      await page.waitForTimeout(200);
      
      // Select an option
      const option = page.locator('select option:nth-child(2)').first();
      if (await option.count() > 0) {
        await option.click();
      }
    }
  });

  test('file upload on mobile', async ({ page }) => {
    await page.goto('/submitticket.php');
    
    const fileInput = page.locator('input[type="file"]');
    
    if (await fileInput.isVisible()) {
      // Check that input is properly sized for touch
      const box = await fileInput.boundingBox();
      if (box) {
        expect(box.height).toBeGreaterThan(30);
      }
    }
  });

  test('date picker works on mobile', async ({ page }) => {
    await page.goto('/admin/clients.php?userid=1');
    
    const dateInput = page.locator('input[type="date"], input.date-picker').first();
    
    if (await dateInput.isVisible()) {
      await dateInput.tap();
      await page.waitForTimeout(500);
      
      // Check if picker appears
      const picker = page.locator('.datepicker, .pika-single, [role="dialog"]');
      const hasPicker = await picker.isVisible().catch(() => false);
      
      expect(hasPicker || (await dateInput.inputValue()).length > 0).toBeTruthy();
    }
  });
});
```

#### Mobile Shopping Tests
```javascript
// mobile-tests/shopping.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Mobile Shopping Experience', () => {
  test.beforeEach(async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
  });

  test('product cards are touch-friendly', async ({ page }) => {
    await page.goto('/cart.php');
    
    const productCards = page.locator('.product-card, .product-item, .item');
    const count = await productCards.count();
    
    if (count > 0) {
      const card = productCards.first();
      const box = await card.boundingBox();
      
      if (box) {
        expect(box.width).toBeLessThanOrEqual(420);
      }
    }
  });

  test('add to cart works on mobile', async ({ page }) => {
    await page.goto('/cart.php');
    
    const addButton = page.locator('.add-to-cart, button.add, .add-btn').first();
    
    if (await addButton.isVisible()) {
      await addButton.tap();
      await page.waitForTimeout(500);
      
      // Check cart updated
      const cartCount = page.locator('.cart-count, .badge');
      if (await cartCount.isVisible()) {
        const text = await cartCount.textContent();
        expect(parseInt(text)).toBeGreaterThan(0);
      }
    }
  });

  test('quantity selector works on mobile', async ({ page }) => {
    await page.goto('/cart.php');
    
    const quantityInput = page.locator('input[type="number"], .quantity input').first();
    
    if (await quantityInput.isVisible()) {
      // Tap and modify quantity
      await quantityInput.tap();
      await quantityInput.fill('2');
      await page.keyboard.press('Done');
    }
  });

  test('checkout process on mobile', async ({ page }) => {
    await page.goto('/cart.php');
    
    const checkoutButton = page.locator('#checkout, .checkout-btn, a[href*="checkout"]').first();
    
    if (await checkoutButton.isVisible()) {
      await checkoutButton.tap();
      await page.waitForTimeout(1000);
      
      // Check we're on checkout page
      const checkoutForm = page.locator('.checkout, #checkout');
      await expect(checkoutForm).toBeVisible({ timeout: 3000 });
    }
  });

  test('swipe gestures work', async ({ page }) => {
    await page.goto('/cart.php');
    
    // Try swipe on cart item
    const cartItem = page.locator('.cart-item').first();
    
    if (await cartItem.count() > 0) {
      const box = await cartItem.boundingBox();
      
      if (box) {
        // Swipe left
        await page.touchscreen.swipe(
          box.x + box.width / 2,
          box.y + box.height / 2,
          box.x - 100,
          box.y + box.height / 2
        );
      }
    }
  });
});
```

### Step 3: Run Mobile Tests
```bash
# Run all mobile tests
npx playwright test --project="iPhone 14 Pro"
npx playwright test --project="Pixel 7"

# Run all mobile projects
npx playwright test -c playwright-mobile.config.js

# Run specific test file
npx playwright test mobile-tests/responsive.spec.js

# Run with debug
npx playwright test --debug mobile-tests/navigation.spec.js
```

### Step 4: Device Lab Integration
```yaml
# .github/workflows/mobile-tests.yml
name: Mobile Tests

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 3 * * *'  # Nightly at 3 AM

jobs:
  mobile-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Run mobile tests
        run: npx playwright test -c playwright-mobile.config.js

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: mobile-test-results
          path: playwright-report/
```

## Mobile Testing Checklist

### Layout & Design
- [ ] Content readable without zooming
- [ ] Touch targets at least 44x44px
- [ ] No horizontal scrolling
- [ ] Images scale properly
- [ ] Font sizes readable

### Navigation
- [ ] Menu accessible on all screens
- [ ] Back button works correctly
- [ ] Breadcrumbs functional
- [ ] Footer navigation works

### Forms
- [ ] Input fields properly sized
- [ ] Keyboard doesn't hide fields
- [ ] Validation messages clear
- [ ] Submit buttons visible after scroll

### Performance
- [ ] Page loads under 3 seconds
- [ ] No layout shifts
- [ ] Smooth scrolling
- [ ] Animations performant

### Device-Specific
- [ ] iOS Safari quirks handled
- [ ] Chrome Android tested
- [ ] Samsung Internet tested
- [ ] Safari iPad tested
