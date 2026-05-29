# WHMCS JavaScript Additions Workflow

## Purpose
Add custom JavaScript functionality to WHMCS without modifying core files.

## Prerequisites
- WHMCS installation
- FTP/cPanel file access
- JavaScript knowledge
- Basic understanding of WHMCS hooks

## Step-by-Step Process

### Step 1: Create Custom JavaScript File
```
1. Navigate to /whmcs/templates/your_template/
2. Create a new file: custom.js
3. Add your JavaScript code
```

### Step 2: Hook JavaScript Into Template

**Option A: Template Hook**
```smarty
{assign var="baseDir" value=$smarty.server.REQUEST_URI|parse_url}
<script src="{$WEB_ROOT}/templates/{$template}/custom.js"></script>
```

**Option B: PHP Hook File**
Create `/whmcs/includes/hooks/custom_js.php`:
```php
<?php

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $jsUrl = rtrim(\WHMCS\Config\Setting::getValue('SystemURL'), '/') 
           . '/templates/' . $vars['template'] . '/custom.js';
    return '<script src="' . $jsUrl . '"></script>';
});
```

### Step 3: Common JavaScript Customizations

**Analytics Tracking:**
```javascript
// Track page views
document.addEventListener('DOMContentLoaded', function() {
    if (typeof gtag !== 'undefined') {
        gtag('config', 'GA_MEASUREMENT_ID', {
            'page_title': document.title,
            'page_location': window.location.href
        });
    }
});
```

**Form Enhancement:**
```javascript
// Auto-save form data
document.addEventListener('DOMContentLoaded', function() {
    const forms = document.querySelectorAll('form[data-autosave]');
    forms.forEach(form => {
        form.addEventListener('input', debounce(function() {
            localStorage.setItem('form_' + form.id, JSON.stringify(
                Object.fromEntries(new FormData(form))
            ));
        }, 1000));
    });
});

function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        clearTimeout(timeout);
        timeout = setTimeout(() => func.apply(this, args), wait);
    };
}
```

**Custom Validation:**
```javascript
// Custom form validation
document.addEventListener('DOMContentLoaded', function() {
    const validateForm = document.getElementById('customForm');
    if (validateForm) {
        validateForm.addEventListener('submit', function(e) {
            const required = this.querySelectorAll('[required]');
            let isValid = true;
            
            required.forEach(field => {
                if (!field.value.trim()) {
                    field.classList.add('is-invalid');
                    isValid = false;
                }
            });
            
            if (!isValid) {
                e.preventDefault();
                alert('Please fill in all required fields.');
            }
        });
    }
});
```

**Interactive Elements:**
```javascript
// Smooth scroll for anchor links
document.addEventListener('DOMContentLoaded', function() {
    document.querySelectorAll('a[href^="#"]').forEach(anchor => {
        anchor.addEventListener('click', function(e) {
            e.preventDefault();
            const target = document.querySelector(this.getAttribute('href'));
            if (target) {
                target.scrollIntoView({ behavior: 'smooth', block: 'start' });
            }
        });
    });
});
```

### Step 4: jQuery Integration (if available)
```javascript
// WHMCS uses jQuery, use noConflict mode
jQuery(document).ready(function($) {
    // Your jQuery code here
    $('.service-list').sortable({
        handle: '.drag-handle',
        update: function() {
            saveServiceOrder();
        }
    });
});
```

### Step 5: AJAX Examples
```javascript
// Custom AJAX call
async function fetchClientData(clientId) {
    try {
        const response = await fetch('ajax.php?action=getClientData&id=' + clientId, {
            headers: {
                'X-Requested-With': 'XMLHttpRequest'
            }
        });
        return await response.json();
    } catch (error) {
        console.error('Error:', error);
        return null;
    }
}
```

### Step 6: Test and Debug
```
1. Open browser DevTools (F12)
2. Check Console tab for errors
3. Use Network tab to verify JS loads
4. Test in incognito mode
```

## Best Practices
- Use 'use strict' mode
- Wrap in DOMContentLoaded or jQuery ready
- Use meaningful variable names
- Minify for production
- Keep code modular
- Handle errors gracefully
- Use semantic versioning for your JS

## Security Considerations
- Never expose sensitive data in JavaScript
- Sanitize user inputs
- Use HTTPS for all external resources
- Validate data on server-side too
- Avoid inline scripts when possible
