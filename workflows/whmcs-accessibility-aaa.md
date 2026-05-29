# WHMCS Accessibility AAA Compliance Workflow

## Purpose
Implement WCAG AAA accessibility standards in WHMCS for maximum accessibility.

## Prerequisites
- WHMCS installation
- WCAG 2.1 AAA knowledge
- CSS/HTML/JavaScript skills
- Accessibility testing tools

## Step-by-Step Process

### Step 1: Color Contrast Requirements (AAA: 7:1)

**Contrast CSS Variables:**
```css
/* WCAG AAA Contrast Ratios (minimum 7:1 for normal text, 4.5:1 for large text) */

:root {
    /* High contrast colors for AAA compliance */
    --contrast-bg: #ffffff;
    --contrast-text: #1a1a1a;  /* ~16:1 contrast */
    --contrast-text-secondary: #4a4a4a;  /* ~7:1 contrast */
    --contrast-link: #0a4b8c;  /* ~7.5:1 contrast */
    --contrast-link-hover: #063566;  /* ~10:1 contrast */
    
    /* Focus indicators */
    --focus-outline: 3px solid #0055cc;
    --focus-offset: 2px;
}

/* Dark mode with high contrast */
@media (prefers-color-scheme: dark) {
    :root {
        --contrast-bg: #0a0a0a;
        --contrast-text: #f0f0f0;  /* ~17:1 contrast */
        --contrast-text-secondary: #c0c0c0;  /* ~7.5:1 contrast */
        --contrast-link: #6db3ff;  /* ~7:1 contrast */
        --contrast-link-hover: #99ccff;  /* ~4.5:1 for hover */
    }
}
```

### Step 2: Enhanced Focus Indicators

```css
/* AAA Compliant Focus States */
*:focus {
    outline: 3px solid #0055cc;
    outline-offset: 2px;
}

*:focus:not(:focus-visible) {
    outline: none;
}

*:focus-visible {
    outline: 3px solid #0055cc;
    outline-offset: 2px;
    box-shadow: 0 0 0 6px rgba(0, 85, 204, 0.3);
}

/* Skip link */
.skip-link {
    position: absolute;
    top: -100px;
    left: 50%;
    transform: translateX(-50%);
    background: var(--contrast-bg);
    color: var(--contrast-link);
    padding: 1rem 2rem;
    z-index: 9999;
    font-weight: 700;
}

.skip-link:focus {
    top: 0;
}
```

### Step 3: Semantic HTML Structure

**ARIA Landmarks:**
```html
<!-- Skip navigation -->
<a href="#main-content" class="skip-link">Skip to main content</a>

<!-- Header with navigation -->
<header role="banner">
    <nav role="navigation" aria-label="Main navigation">
        <!-- Navigation items -->
    </nav>
</header>

<!-- Main content -->
<main id="main-content" role="main" tabindex="-1">
    <!-- Page content -->
</main>

<!-- Complementary content -->
<aside role="complementary" aria-label="Related information">
    <!-- Sidebar content -->
</aside>

<!-- Footer -->
<footer role="contentinfo">
    <!-- Footer content -->
</footer>
```

### Step 4: Form Accessibility

```html
<!-- Form with proper labeling -->
<form action="/action" method="post">
    <div class="form-group">
        <label for="first-name">
            First Name
            <span class="required" aria-hidden="true">*</span>
            <span class="visually-hidden">(required)</span>
        </label>
        <input type="text" 
               id="first-name" 
               name="firstname" 
               required
               aria-required="true"
               aria-describedby="first-name-help first-name-error">
        <small id="first-name-help" class="help-text">Enter your first name as shown on your ID</small>
        <span id="first-name-error" class="error-message" role="alert"></span>
    </div>
</form>
```

**Error Message Patterns:**
```javascript
// Accessible error handling
function showError(inputId, message) {
    const input = document.getElementById(inputId);
    const errorId = inputId + '-error';
    let errorElement = document.getElementById(errorId);
    
    if (!errorElement) {
        errorElement = document.createElement('span');
        errorElement.id = errorId;
        errorElement.className = 'error-message';
        errorElement.setAttribute('role', 'alert');
        input.parentNode.appendChild(errorElement);
    }
    
    input.setAttribute('aria-invalid', 'true');
    input.setAttribute('aria-describedby', errorId);
    errorElement.textContent = message;
}

function clearError(inputId) {
    const input = document.getElementById(inputId);
    const errorId = inputId + '-error';
    const errorElement = document.getElementById(errorId);
    
    if (errorElement) {
        errorElement.remove();
    }
    
    input.removeAttribute('aria-invalid');
}
```

### Step 5: Heading Structure (AAA)

```html
<!-- Proper heading hierarchy -->
<header>
    <h1>Company Name</h1>
</header>

<main>
    <article>
        <h2>Page Title</h2>
        <section>
            <h3>Section 1</h3>
            <h4>Sub-section 1.1</h4>
        </section>
        <section>
            <h3>Section 2</h3>
            <h4>Sub-section 2.1</h4>
        </section>
    </article>
</main>

<!-- Navigation with proper headings -->
<nav aria-label="Breadcrumb">
    <h2 class="visually-hidden">You are here</h2>
    <ol>
        <li><a href="/">Home</a></li>
        <li><a href="/services">Services</a></li>
        <li aria-current="page">Current Page</li>
    </ol>
</nav>
```

### Step 6: Keyboard Navigation Enhancements

```css
/* Enhanced keyboard focus */
.enhanced-focus *:focus {
    outline: 3px solid #0055cc !important;
    outline-offset: 2px !important;
}

/* Visible focus for all interactive elements */
button:focus-visible,
a:focus-visible,
input:focus-visible,
select:focus-visible,
textarea:focus-visible,
[tabindex]:focus-visible {
    outline: 3px solid #0055cc;
    outline-offset: 2px;
    box-shadow: 0 0 0 6px rgba(0, 85, 204, 0.3);
}

/* Visible focus on mouse click too */
button:focus:not(:focus-visible),
a:focus:not(:focus-visible) {
    outline: none;
}
```

```javascript
// Trap focus in modals
function trapFocus(element) {
    const focusableElements = element.querySelectorAll(
        'a[href], button:not([disabled]), textarea:not([disabled]), input:not([disabled]), select:not([disabled]), [tabindex]:not([tabindex="-1"])'
    );
    
    const firstFocusable = focusableElements[0];
    const lastFocusable = focusableElements[focusableElements.length - 1];
    
    element.addEventListener('keydown', function(e) {
        if (e.key === 'Tab') {
            if (e.shiftKey) {
                if (document.activeElement === firstFocusable) {
                    e.preventDefault();
                    lastFocusable.focus();
                }
            } else {
                if (document.activeElement === lastFocusable) {
                    e.preventDefault();
                    firstFocusable.focus();
                }
            }
        }
        
        if (e.key === 'Escape') {
            closeModal(element);
        }
    });
}
```

### Step 7: Screen Reader Only Content

```css
/* Visually hidden but accessible to screen readers */
.visually-hidden,
.sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
}

/* Show on focus for skip links */
.visually-hidden.focusable:active,
.visually-hidden.focusable:focus {
    position: static;
    width: auto;
    height: auto;
    margin: 0;
    overflow: visible;
    clip: auto;
    white-space: normal;
}

/* Hidden but available */
[hidden] {
    display: none !important;
}
```

### Step 8: Dynamic Content Announcements

```javascript
// Live region for announcements
function announceToScreenReader(message, priority = 'polite') {
    let liveRegion = document.getElementById('live-announcer');
    
    if (!liveRegion) {
        liveRegion = document.createElement('div');
        liveRegion.id = 'live-announcer';
        liveRegion.setAttribute('aria-live', priority);
        liveRegion.setAttribute('aria-atomic', 'true');
        liveRegion.className = 'visually-hidden';
        document.body.appendChild(liveRegion);
    }
    
    // Clear and set message (triggers announcement)
    liveRegion.textContent = '';
    setTimeout(() => {
        liveRegion.textContent = message;
    }, 100);
}

// Usage examples
announceToScreenReader('Form submitted successfully', 'assertive');
announceToScreenReader('3 items in your cart', 'polite');
announceToScreenReader('Navigation menu opened', 'polite');
```

### Step 9: Link Text Best Practices

```html
<!-- Good: Descriptive link text -->
<a href="/article/web-accessibility">Learn more about web accessibility guidelines</a>

<!-- Bad: Non-descriptive link text -->
<a href="/article/web-accessibility">Click here</a>
<a href="/article/web-accessibility">Read more</a>

<!-- Links that open in new tab -->
<a href="https://example.com" target="_blank" rel="noopener noreferrer">
    External website (opens in new tab)
</a>

<!-- Download links -->
<a href="/files/document.pdf" download>
    Download Annual Report 2024 (PDF, 2.5 MB)
</a>
```

### Step 10: Table Accessibility

```html
<table>
    <caption>Service Comparison - Monthly Pricing</caption>
    <thead>
        <tr>
            <th scope="col">Feature</th>
            <th scope="col">Basic</th>
            <th scope="col">Professional</th>
            <th scope="col">Enterprise</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <th scope="row">Storage</th>
            <td>10 GB</td>
            <td>50 GB</td>
            <td>Unlimited</td>
        </tr>
    </tbody>
</table>
```

### Step 11: Image Accessibility

```html
<!-- Informative image -->
<img src="server-diagram.jpg" 
     alt="Diagram showing server architecture with load balancer, application servers, and database cluster">

<!-- Decorative image -->
<img src="decorative-line.png" alt="" role="presentation">

<!-- Complex image -->
<img src="chart.png" alt="Bar chart showing sales growth">
<details>
    <summary>Read data table</summary>
    <table>
        <!-- Data table for screen readers -->
    </table>
</details>
```

### Step 12: Touch Target Sizing (AAA: 44x44px minimum)

```css
/* AAA compliant touch targets */
button,
a,
input[type="checkbox"],
input[type="radio"],
select,
[role="button"],
[role="menuitem"],
[role="tab"] {
    min-height: 44px;
    min-width: 44px;
    padding: 0.75rem 1rem;
}

/* Touch-friendly spacing */
.interactive-element {
    margin-bottom: 0.5rem;
}

/* Ensure adequate spacing between targets */
.buttons-group button {
    margin: 0.25rem;
}
```

### Step 13: Reduced Motion

```css
/* Respect user preference */
@media (prefers-reduced-motion: reduce) {
    *,
    *::before,
    *::after {
        animation-duration: 0.01ms !important;
        animation-iteration-count: 1 !important;
        transition-duration: 0.01ms !important;
        scroll-behavior: auto !important;
    }
}

/* JavaScript check */
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)');

function shouldAnimate() {
    return !prefersReducedMotion.matches;
}
```

### Step 14: Accessibility Testing Hook

```php
<?php
/**
 * Add accessibility features
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return '
    <style>
        .visually-hidden {
            position: absolute;
            width: 1px;
            height: 1px;
            padding: 0;
            margin: -1px;
            overflow: hidden;
            clip: rect(0, 0, 0, 0);
            white-space: nowrap;
            border: 0;
        }
    </style>
    ';
});
```

### Step 15: Automated Testing Integration

```javascript
// Accessibility testing function
async function runAccessibilityAudit() {
    console.log('Running accessibility audit...');
    
    // Check for common issues
    const issues = [];
    
    // Images without alt text
    document.querySelectorAll('img:not([alt])').forEach(img => {
        issues.push({ element: img, issue: 'Image missing alt attribute' });
    });
    
    // Buttons without text
    document.querySelectorAll('button:empty').forEach(btn => {
        issues.push({ element: btn, issue: 'Button has no text content' });
    });
    
    // Links without text
    document.querySelectorAll('a:not([aria-label]):empty').forEach(link => {
        issues.push({ element: link, issue: 'Link has no text or aria-label' });
    });
    
    // Form inputs without labels
    document.querySelectorAll('input:not([id])').forEach(input => {
        issues.push({ element: input, issue: 'Input missing id attribute' });
    });
    
    return issues;
}
```

## Best Practices
- Maintain 7:1 contrast ratio for AAA compliance
- Provide text alternatives for all non-text content
- Create content that can be presented in different ways
- Make it easier for users to see content
- Make it easier to use with keyboard
- Provide ways to help users navigate
- Make text readable and understandable
- Help users avoid mistakes
- Support compatibility with current and future tools
- Test with actual screen readers (NVDA, VoiceOver, JAWS)
- Include users with disabilities in testing
- Document accessibility features
