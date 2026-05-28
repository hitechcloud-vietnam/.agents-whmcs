# WHMCS AJAX Patterns Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing AJAX functionality in WHMCS modules.

## When to Use

- Building interactive admin pages
- Real-time status updates
- Async form submissions

## AJAX Patterns

### Admin AJAX Endpoint

```php
<?php
// modules/addons/{module}/ajax.php
if (!defined("WHMCS")) { die("Direct access denied"); }

// Check admin auth
if (!checkAuth()) {
    jsonResponse(['error' => 'Unauthorized'], 401);
}

// Get action
$action = $_REQUEST['action'] ?? '';

switch ($action) {
    case 'get_stats':
        jsonResponse(getStats());
        break;
    case 'save_data':
        jsonResponse(saveData($_POST));
        break;
    case 'delete_item':
        jsonResponse(deleteItem($_POST['id']));
        break;
    default:
        jsonResponse(['error' => 'Unknown action'], 400);
}

function jsonResponse(array $data, int $status = 200): void {
    http_response_code($status);
    header('Content-Type: application/json');
    echo json_encode($data);
    exit;
}
```

### JavaScript AJAX Client

```javascript
// assets/js/module.js
const ModuleAPI = {
    async request(action, data = {}) {
        try {
            const response = await fetch('ajax.php?action=' + action, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'X-Requested-With': 'XMLHttpRequest',
                },
                body: JSON.stringify(data)
            });

            return await response.json();
        } catch (error) {
            console.error('API Error:', error);
            return { error: 'Request failed' };
        }
    },

    async getStats() {
        return this.request('get_stats');
    },

    async saveData(data) {
        return this.request('save_data', data);
    }
};

// Usage
async function loadStats() {
    const result = await ModuleAPI.getStats();
    if (result.error) {
        showError(result.error);
    } else {
        updateDisplay(result);
    }
}
```

### AJAX with CSRF Token

```javascript
// Include token in all AJAX requests
async function ajaxWithToken(action, data) {
    const token = document.querySelector('input[name="token"]')?.value;

    const response = await fetch('ajax.php?action=' + action, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': token
        },
        body: JSON.stringify(data)
    });

    return response.json();
}

// Template includes token
/*
<input type="hidden" name="token" value="{$token}">
*/

// Server validates token
function validateAjaxToken(): bool {
    $token = $_SERVER['HTTP_X_CSRF_TOKEN'] ?? $_POST['token'] ?? '';
    return check_token('WHMCS.admin.default', false, $token);
}
```

## Checklist

- [ ] AJAX endpoint file
- [ ] JSON response format
- [ ] Authentication check
- [ ] CSRF token validation
- [ ] Error handling
- [ ] Loading states

---

**Related Skills:**
- whmcs-clientarea-builder
- whmcs-template-styling
- whmcs-security-hardening