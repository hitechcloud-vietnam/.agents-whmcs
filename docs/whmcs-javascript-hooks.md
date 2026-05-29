# WHMCS JavaScript Hooks

## Overview

JavaScript hooks in WHMCS allow client-side interactivity and integration with third-party scripts. They enable AJAX functionality, form validation, and dynamic content updates.

## Including JavaScript

### In Templates

```smarty
{include file="$template/includes/scripts.tpl"}

{block name="scripts"}
    <script src="{$base_path}/js/custom.js"></script>
{/block}
```

### Via Hook

```php
<?php
add_hook('ClientAreaFooterOutput', 1, function($vars) {
    return '<script src="custom.js"></script>';
});
```

### Inline Script

```php
<?php
add_hook('ClientAreaFooterOutput', 1, function($vars) {
    return '<script>console.log("Custom script loaded");</script>';
});
```

## Document Ready

### jQuery Ready (jQuery Available)

```javascript
jQuery(document).ready(function($) {
    // Your code here
    console.log('Page loaded');
});
```

### Vanilla JavaScript

```javascript
document.addEventListener('DOMContentLoaded', function() {
    console.log('DOM ready');
});
```

## Common WHMCS JavaScript Functions

### WHMCS.* Functions

```javascript
// Display error message
WHMCS.UI.notify.error('Error message');

// Display success message
WHMRS.UI.notify.success('Success message');

// Display info message
WHMCS.UI.notify.info('Info message');

// Confirm dialog
WHMCS.UI.confirm('Are you sure?', function() {
    console.log('Confirmed');
});

// Redirect
WHMCS.redirect('clientarea.php');

// Form validation
if (!WHMCS.validate('#form-id')) {
    return false;
}
```

### jQuery AJAX Helpers

```javascript
// AJAX GET request
WHMC.ajax({
    type: 'GET',
    url: 'ajax.php',
    data: { action: 'getData' },
    success: function(response) {
        console.log(response);
    }
});

// AJAX POST request
$.post('ajax.php', {
    action: 'submitForm',
    data: $('#form').serialize()
}, function(response) {
    if (response.success) {
        WHMCS.UI.notify.success('Saved successfully');
    }
});
```

## Form Handling

### Form Validation

```javascript
jQuery(document).ready(function($) {
    // Required field validation
    $('#registration-form').on('submit', function(e) {
        var isValid = true;
        
        $(this).find('[required]').each(function() {
            if (!$(this).val()) {
                $(this).addClass('is-invalid');
                isValid = false;
            } else {
                $(this).removeClass('is-invalid');
            }
        });
        
        if (!isValid) {
            e.preventDefault();
            WHMCS.UI.notify.error('Please fill all required fields');
        }
    });
    
    // Email validation
    $('#email').on('blur', function() {
        var email = $(this).val();
        var emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        
        if (!emailRegex.test(email)) {
            $(this).addClass('is-invalid');
            $(this).next('.invalid-feedback').text('Invalid email format');
        } else {
            $(this).removeClass('is-invalid');
        }
    });
});
```

### AJAX Form Submission

```javascript
jQuery(document).ready(function($) {
    $('#contact-form').on('submit', function(e) {
        e.preventDefault();
        
        var $form = $(this);
        var $btn = $form.find('button[type="submit"]');
        
        // Disable button
        $btn.prop('disabled', true).text('Sending...');
        
        $.ajax({
            url: 'ajax.php',
            type: 'POST',
            data: $form.serialize(),
            dataType: 'json',
            success: function(response) {
                if (response.success) {
                    WHMCS.UI.notify.success('Message sent successfully');
                    $form[0].reset();
                } else {
                    WHMCS.UI.notify.error(response.error || 'An error occurred');
                }
            },
            error: function() {
                WHMCS.UI.notify.error('Network error. Please try again.');
            },
            complete: function() {
                $btn.prop('disabled', false).text('Send Message');
            }
        });
    });
});
```

## Cart Operations

### Add to Cart AJAX

```javascript
jQuery(document).ready(function($) {
    $('.add-to-cart').on('click', function(e) {
        e.preventDefault();
        
        var $btn = $(this);
        var pid = $btn.data('pid');
        var cycle = $btn.data('cycle') || 'monthly';
        
        $btn.prop('disabled', true).html('<i class="fa fa-spinner fa-spin"></i>');
        
        $.post('cart.php', {
            a: 'add',
            pid: pid,
            billingcycle: cycle
        }, function(response) {
            // Update cart count
            updateCartCount();
            WHMCS.UI.notify.success('Added to cart');
        }).fail(function() {
            WHMCS.UI.notify.error('Failed to add item');
        }).always(function() {
            $btn.prop('disabled', false).text('Add to Cart');
        });
    });
});

function updateCartCount() {
    $.get('cart.php', { a: 'count' }, function(count) {
        $('.cart-count').text(count);
    });
}
```

### Quantity Update

```javascript
jQuery(document).ready(function($) {
    $('.quantity-input').on('change', function() {
        var $input = $(this);
        var itemId = $input.data('item-id');
        var quantity = $input.val();
        
        $.post('cart.php', {
            a: 'update',
            id: itemId,
            qty: quantity
        }, function() {
            // Update totals
            updateCartTotals();
        });
    });
});
```

## Service Management

### Service Actions

```javascript
jQuery(document).ready(function($) {
    // Upgrade service
    $('.upgrade-service').on('click', function(e) {
        e.preventDefault();
        
        var serviceId = $(this).data('service-id');
        
        WHMCS.UI.confirm('Upgrade this service?', function() {
            $.post('clientarea.php', {
                action: 'upgradeService',
                serviceId: serviceId
            }, function(response) {
                if (response.success) {
                    location.reload();
                }
            });
        });
    });
    
    // Cancel service
    $('.cancel-service').on('click', function(e) {
        e.preventDefault();
        
        var serviceId = $(this).data('service-id');
        
        WHMCS.UI.confirm('Cancel this service? This cannot be undone.', function() {
            $.post('clientarea.php', {
                action: 'cancelService',
                serviceId: serviceId
            }, function(response) {
                if (response.success) {
                    location.reload();
                }
            });
        });
    });
});
```

## Custom Events

### Hook into WHMCS Events

```javascript
jQuery(document).ready(function($) {
    // After page load
    $(document).on('whmcs.page.loaded', function() {
        console.log('WHMCS page loaded');
    });
    
    // After AJAX operations
    $(document).on('whmcs.ajax.complete', function(event, xhr, settings) {
        console.log('AJAX complete:', settings.url);
    });
    
    // On notification shown
    $(document).on('whmcs.notify.shown', function(event, notification) {
        console.log('Notification:', notification);
    });
});
```

## Client Area Specific

### Tab Switching

```javascript
jQuery(document).ready(function($) {
    $('.client-tabs a').on('click', function(e) {
        e.preventDefault();
        
        var target = $(this).attr('href');
        
        $('.client-tabs a').removeClass('active');
        $(this).addClass('active');
        
        $('.tab-content > div').removeClass('active');
        $(target).addClass('active');
    });
});
```

### Accordion

```javascript
jQuery(document).ready(function($) {
    $('.accordion-header').on('click', function() {
        var $header = $(this);
        var $content = $header.next('.accordion-content');
        
        $header.toggleClass('active');
        $content.slideToggle(300);
        
        // Close others
        $('.accordion-header').not($header).removeClass('active');
        $('.accordion-content').not($content).slideUp(300);
    });
});
```

## Third-Party Integration

### Google Analytics

```javascript
jQuery(document).ready(function($) {
    // Track page views
    ga('send', 'pageview', {
        'page': window.location.pathname,
        'title': document.title
    });
    
    // Track events
    $('.btn-purchase').on('click', function() {
        ga('send', 'event', 'Button', 'Click', 'Purchase');
    });
});
```

### Facebook Pixel

```javascript
jQuery(document).ready(function($) {
    // Track AddToCart
    $('.add-to-cart').on('click', function() {
        fbq('track', 'AddToCart', {
            content_name: $(this).data('product-name'),
            content_category: $(this).data('category'),
            value: $(this).data('price'),
            currency: 'USD'
        });
    });
    
    // Track Purchase
    $('.checkout-complete').on('click', function() {
        fbq('track', 'Purchase', {
            value: $(this).data('total'),
            currency: 'USD'
        });
    });
});
```

## Best Practices

1. **Use jQuery** - WHMCS includes jQuery
2. **DOM Ready** - Wait for DOMContentLoaded
3. **Event Delegation** - For dynamic content
4. **Error Handling** - Always handle AJAX errors
5. **Non-Blocking** - Don't block UI thread

## See Also

- [AJAX Templates](../whmcs-ajax-templates.md)
- [Template Hooks](../whmcs-template-hooks.md)
- [Template Filters](../whmcs-template-filters.md)