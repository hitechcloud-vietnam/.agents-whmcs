# WHMCS JavaScript & AJAX

## Overview

JavaScript and AJAX enable dynamic, interactive functionality in WHMCS.

## Basic JavaScript

### Variable Declaration

```javascript
// Variables
let clientId = 123;
const apiUrl = '/api/endpoint';

// Objects
const client = {
    id: 123,
    name: 'John Doe',
    email: 'john@example.com'
};

// Arrays
const products = ['Product 1', 'Product 2', 'Product 3'];
```

### DOM Manipulation

```javascript
// Select elements
const element = document.getElementById('element-id');
const elements = document.querySelectorAll('.class-name');

// Modify content
element.textContent = 'New text';
element.innerHTML = '<strong>Bold text</strong>';

// Modify styles
element.style.color = '#0066cc';
element.classList.add('active');
element.classList.remove('hidden');

// Event listeners
element.addEventListener('click', function(e) {
    console.log('Clicked!');
});
```

## jQuery Usage

```javascript
// Document ready
$(document).ready(function() {
    // Code here
});

// Selectors
$('#element-id');           // By ID
$('.class-name');            // By class
$('input[name="email"]');    // By attribute
$('tr:even');                // By filter

// Events
$('#submit-btn').on('click', function(e) {
    e.preventDefault();
    // Handle click
});

// AJAX
$.ajax({
    url: '/api/endpoint',
    type: 'POST',
    data: { id: 123 },
    success: function(response) {
        console.log(response);
    },
    error: function(xhr, status, error) {
        console.error(error);
    }
});

// Get form data
const formData = $('#my-form').serialize();

// JSON AJAX
$.getJSON('/api/endpoint', { id: 123 }, function(data) {
    console.log(data);
});
```

## AJAX Implementation

### Basic AJAX Class

```javascript
class WHMCSApiClient {
    constructor(baseUrl) {
        this.baseUrl = baseUrl;
    }
    
    async request(action, params = {}) {
        const url = this.baseUrl + '/includes/api.php';
        
        const data = {
            action: action,
            identifier: this.apiIdentifier,
            secret: this.apiSecret,
            responsetype: 'json',
            ...params
        };
        
        try {
            const response = await fetch(url, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded',
                },
                body: new URLSearchParams(data)
            });
            
            return await response.json();
        } catch (error) {
            console.error('API Error:', error);
            throw error;
        }
    }
    
    async getClients(params = {}) {
        return this.request('GetClients', params);
    }
    
    async getClient(clientId) {
        return this.request('GetClient', { clientid: clientId });
    }
    
    async updateClient(clientId, data) {
        return this.request('UpdateClient', { clientid: clientId, ...data });
    }
}

// Usage
const api = new WHMCSApiClient('https://whmcs.example.com');
const clients = await api.getClients({ limitnum: 100 });
```

### AJAX Form Submission

```javascript
$(document).ready(function() {
    $('#submit-form').on('submit', function(e) {
        e.preventDefault();
        
        const $form = $(this);
        const $btn = $form.find('button[type="submit"]');
        const originalText = $btn.text();
        
        // Disable button
        $btn.prop('disabled', true).text('Processing...');
        
        // Get form data
        const formData = new FormData(this);
        
        $.ajax({
            url: $form.attr('action'),
            type: 'POST',
            data: formData,
            processData: false,
            contentType: false,
            success: function(response) {
                if (response.success) {
                    showMessage('success', response.message);
                    // Redirect or update UI
                } else {
                    showMessage('error', response.message);
                }
            },
            error: function(xhr, status, error) {
                showMessage('error', 'An error occurred. Please try again.');
            },
            complete: function() {
                $btn.prop('disabled', false).text(originalText);
            }
        });
    });
});

function showMessage(type, message) {
    const alertClass = type === 'success' ? 'alert-success' : 'alert-danger';
    const html = `<div class="alert ${alertClass} alert-dismissible fade show" role="alert">
        ${message}
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>`;
    
    $('#messages').html(html);
    setTimeout(() => $('.alert').alert('close'), 5000);
}
```

### Async/Await Pattern

```javascript
class DataTableManager {
    constructor(config) {
        this.config = config;
        this.page = 1;
        this.perPage = config.perPage || 25;
        this.sortColumn = config.sortColumn || 'id';
        this.sortDirection = config.sortDirection || 'asc';
    }
    
    async loadData() {
        try {
            this.showLoading();
            
            const response = await $.ajax({
                url: this.config.apiUrl,
                type: 'GET',
                data: {
                    page: this.page,
                    limitnum: this.perPage,
                    orderby: this.sortColumn,
                    sort: this.sortDirection
                }
            });
            
            this.renderTable(response.data);
            this.renderPagination(response);
            
        } catch (error) {
            console.error('Failed to load data:', error);
            this.showError('Failed to load data');
        } finally {
            this.hideLoading();
        }
    }
    
    renderTable(data) {
        const rows = data.map(item => `
            <tr>
                <td>${item.id}</td>
                <td>${item.name}</td>
                <td>${item.email}</td>
                <td>${item.status}</td>
            </tr>
        `).join('');
        
        this.config.tableBody.html(rows);
    }
}
```

## Module JavaScript

### Client-Side Module

```javascript
// modules/addons/yourmodule/assets/js/module.js

(function() {
    'use strict';
    
    const YourModule = {
        config: {
            baseUrl: '',
            csrfToken: '',
        },
        
        init: function(config) {
            this.config = Object.assign(this.config, config);
            this.bindEvents();
        },
        
        bindEvents: function() {
            // Button clicks
            $(document).on('click', '.js-your-action', this.handleAction.bind(this));
            
            // Form submissions
            $(document).on('submit', '.js-your-form', this.handleFormSubmit.bind(this));
            
            // Dynamic events
            $(document).on('change', '.js-your-select', this.handleSelectChange.bind(this));
        },
        
        handleAction: function(e) {
            const $btn = $(e.currentTarget);
            const action = $btn.data('action');
            const id = $btn.data('id');
            
            if (confirm('Are you sure?')) {
                this.performAction(action, { id: id });
            }
        },
        
        handleFormSubmit: function(e) {
            e.preventDefault();
            
            const $form = $(e.currentTarget);
            const formData = $form.serialize();
            
            this.submitForm($form.attr('action'), formData);
        },
        
        handleSelectChange: function(e) {
            const value = $(e.currentTarget).val();
            this.filterData(value);
        },
        
        async performAction(action, params) {
            try {
                const response = await this.ajax({
                    url: this.config.baseUrl + '/modules/addons/yourmodule/api.php',
                    data: { action: action, ...params }
                });
                
                if (response.success) {
                    this.showNotification('success', response.message);
                    this.refreshData();
                } else {
                    this.showNotification('error', response.message);
                }
            } catch (error) {
                this.showNotification('error', 'An error occurred');
            }
        },
        
        ajax: function(options) {
            return $.ajax({
                url: options.url,
                type: 'POST',
                data: {
                    ...options.data,
                    token: this.config.csrfToken
                }
            });
        },
        
        showNotification: function(type, message) {
            // Implementation
        },
        
        refreshData: function() {
            // Implementation
        }
    };
    
    // Expose to global scope
    window.YourModule = YourModule;
    
})();
```

## Best Practices

1. **Use HTTPS** - Always use secure connections
2. **Handle errors** - Show user-friendly error messages
3. **Show loading states** - Indicate pending operations
4. **Validate data** - Check response before using
5. **Use CSRF tokens** - Prevent CSRF attacks

## Related Documentation

- [WHMCS Form Handling](/docs/whmcs-form-handling.md)
- [WHMCS Data Tables](/docs/whmcs-data-tables.md)