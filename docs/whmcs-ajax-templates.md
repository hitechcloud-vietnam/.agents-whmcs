# WHMCS AJAX Template Patterns

## Overview

AJAX patterns in WHMCS enable dynamic content loading without page refreshes. This improves user experience and performance for cart, checkout, and service management operations.

## AJAX Basics in WHMCS

### Basic AJAX Structure

```javascript
// Standard AJAX call
jQuery.ajax({
    url: 'ajax.php',
    type: 'POST',
    data: {
        action: 'customAction',
        token: csrfToken()
    },
    dataType: 'json',
    success: function(response) {
        if (response.success) {
            // Handle success
        } else {
            // Handle error
        }
    },
    error: function(xhr, status, error) {
        // Handle error
    }
});
```

### CSRF Token

```javascript
function csrfToken() {
    return jQuery('input[name="token"]').val() || 
           jQuery('meta[name="csrf-token"]').attr('content');
}

jQuery.ajaxSetup({
    beforeSend: function(xhr, settings) {
        xhr.setRequestHeader('X-CSRF-TOKEN', csrfToken());
    }
});
```

## AJAX Endpoints

### Standard WHMCS AJAX Files

```
ajax.php                    // Main AJAX endpoint
includes/api.php            // API endpoint
includes/json.php           // JSON responses
```

### Custom AJAX Handler

```php
<?php
// custom-ajax.php
define('WHMCS', true);
require_once __DIR__ . '/../init.php';

header('Content-Type: application/json');

$action = $_REQUEST['action'] ?? '';
$response = ['success' => false];

switch ($action) {
    case 'getProductDetails':
        $pid = (int)($_REQUEST['pid'] ?? 0);
        $product = \WHMCS\Product\Product::getProduct($pid);
        $response = [
            'success' => true,
            'data' => [
                'name' => $product->name,
                'price' => $product->pricing()->first()->monthlyPrice(),
                'description' => $product->description
            ]
        ];
        break;
        
    default:
        $response['error'] = 'Unknown action';
}

echo json_encode($response);
```

## Cart AJAX

### Add to Cart

```javascript
jQuery(document).ready(function($) {
    $('.product-add-btn').on('click', function() {
        var $btn = $(this);
        var pid = $btn.data('pid');
        var cycle = $btn.data('cycle') || 'monthly';
        
        $btn.prop('disabled', true)
            .html('<span class="spinner"></span> Adding...');
        
        $.ajax({
            url: 'cart.php?a=add',
            type: 'POST',
            data: {
                pid: pid,
                billingcycle: cycle
            },
            success: function(response) {
                updateCartWidget();
                WHMCS.UI.notify.success('Added to cart');
            },
            error: function() {
                WHMCS.UI.notify.error('Failed to add item');
            },
            complete: function() {
                $btn.prop('disabled', false).text('Add to Cart');
            }
        });
    });
});
```

### Update Cart

```javascript
function updateCartItem(itemId, data) {
    return $.ajax({
        url: 'cart.php?a=update',
        type: 'POST',
        data: {
            id: itemId,
            ...data
        },
        dataType: 'json'
    });
}

jQuery(document).ready(function($) {
    // Billing cycle change
    $('[name="billingcycle"]').on('change', function() {
        var itemId = $(this).data('item-id');
        var cycle = $(this).val();
        
        updateCartItem(itemId, { billingcycle: cycle })
            .done(function(response) {
                updateCartTotals();
            });
    });
    
    // Quantity change
    $('[name="qty"]').on('change', function() {
        var itemId = $(this).data('item-id');
        var qty = $(this).val();
        
        updateCartItem(itemId, { qty: qty })
            .done(function(response) {
                updateCartTotals();
            });
    });
});
```

### Remove from Cart

```javascript
function removeCartItem(itemId) {
    return $.ajax({
        url: 'cart.php?a=remove',
        type: 'POST',
        data: { id: itemId }
    });
}

jQuery(document).ready(function($) {
    $('.remove-item').on('click', function() {
        var $row = $(this).closest('tr');
        var itemId = $(this).data('item-id');
        
        WHMCS.UI.confirm('Remove this item?', function() {
            removeCartItem(itemId)
                .done(function() {
                    $row.fadeOut(300, function() {
                        $(this).remove();
                        updateCartTotals();
                    });
                });
        });
    });
});
```

## Dynamic Content Loading

### Load More Pattern

```javascript
jQuery(document).ready(function($) {
    var page = 1;
    var loading = false;
    var hasMore = true;
    
    $(window).on('scroll', function() {
        if (loading || !hasMore) return;
        
        if ($(window).scrollTop() + $(window).height() 
            >= $(document).height() - 200) {
            loadMoreItems();
        }
    });
    
    function loadMoreItems() {
        loading = true;
        $('#loading').show();
        
        $.ajax({
            url: 'ajax.php',
            data: {
                action: 'loadMore',
                type: 'services',
                page: page
            },
            success: function(response) {
                if (response.items.length > 0) {
                    $('#item-list').append(response.html);
                    page++;
                } else {
                    hasMore = false;
                }
            },
            complete: function() {
                loading = false;
                $('#loading').hide();
            }
        });
    }
});
```

### Search with Debounce

```javascript
jQuery(document).ready(function($) {
    var searchTimeout;
    
    $('#search-input').on('input', function() {
        var query = $(this).val();
        
        clearTimeout(searchTimeout);
        
        if (query.length < 3) {
            $('#search-results').hide();
            return;
        }
        
        searchTimeout = setTimeout(function() {
            $.ajax({
                url: 'ajax.php',
                data: {
                    action: 'search',
                    q: query
                },
                success: function(response) {
                    displaySearchResults(response.results);
                }
            });
        }, 300);
    });
    
    function displaySearchResults(results) {
        var $results = $('#search-results');
        
        if (results.length === 0) {
            $results.html('<p>No results found</p>').show();
            return;
        }
        
        var html = results.map(function(item) {
            return '<div class="result-item">' +
                   '<a href="' + item.url + '">' + item.name + '</a>' +
                   '</div>';
        }).join('');
        
        $results.html(html).show();
    }
});
```

## Form Validation

### Real-time Validation

```javascript
jQuery(document).ready(function($) {
    var validationRules = {
        email: {
            pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
            message: 'Please enter a valid email'
        },
        phone: {
            pattern: /^[\d\s\-\+\(\)]{10,}$/,
            message: 'Please enter a valid phone number'
        }
    };
    
    $('[data-validate]').on('blur', function() {
        var $input = $(this);
        var rule = validationRules[$input.data('validate')];
        
        if (rule && !rule.pattern.test($input.val())) {
            showValidationError($input, rule.message);
        } else {
            clearValidationError($input);
        }
    });
    
    function showValidationError($input, message) {
        $input.addClass('is-invalid')
               .next('.invalid-feedback').remove();
        $input.after('<div class="invalid-feedback">' + message + '</div>');
    }
    
    function clearValidationError($input) {
        $input.removeClass('is-invalid')
               .next('.invalid-feedback').remove();
    }
});
```

## Service Management

### Status Updates

```javascript
jQuery(document).ready(function($) {
    $('.refresh-status').on('click', function() {
        var $btn = $(this);
        var serviceId = $btn.data('service-id');
        
        $btn.find('i').addClass('fa-spin');
        
        $.ajax({
            url: 'clientarea.php',
            data: {
                action: 'getServiceStatus',
                serviceId: serviceId
            },
            success: function(response) {
                updateServiceStatus(serviceId, response.status);
            },
            complete: function() {
                $btn.find('i').removeClass('fa-spin');
            }
        });
    });
});
```

### Password Generation

```javascript
jQuery(document).ready(function($) {
    $('.generate-password').on('click', function() {
        $.ajax({
            url: 'clientarea.php',
            data: {
                action: 'generatePassword',
                length: 16
            },
            success: function(response) {
                $('#password-input').val(response.password);
            }
        });
    });
});
```

## Modal Windows

### AJAX Modal

```javascript
jQuery(document).ready(function($) {
    $('.open-modal').on('click', function(e) {
        e.preventDefault();
        
        var $btn = $(this);
        var url = $btn.attr('href') || $btn.data('url');
        var title = $btn.data('title') || 'Modal';
        
        var $modal = $('#ajax-modal');
        $modal.find('.modal-title').text(title);
        $modal.find('.modal-body').html('<div class="text-center"><i class="fa fa-spinner fa-spin fa-3x"></i></div>');
        $modal.modal('show');
        
        $.ajax({
            url: url,
            success: function(html) {
                $modal.find('.modal-body').html(html);
            }
        });
    });
});
```

### Modal Template

```html
<div class="modal fade" id="ajax-modal" tabindex="-1" role="dialog">
    <div class="modal-dialog" role="document">
        <div class="modal-content">
            <div class="modal-header">
                <button type="button" class="close" data-dismiss="modal">
                    <span>&times;</span>
                </button>
                <h4 class="modal-title">Modal Title</h4>
            </div>
            <div class="modal-body">
                <!-- AJAX content here -->
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-default" data-dismiss="modal">Close</button>
                <button type="button" class="btn btn-primary">Save</button>
            </div>
        </div>
    </div>
</div>
```

## Error Handling

### Global AJAX Error Handler

```javascript
jQuery(document).ajaxError(function(event, xhr, settings, error) {
    if (xhr.status === 401) {
        // Unauthorized - redirect to login
        window.location.href = 'login.php';
    } else if (xhr.status === 403) {
        // Forbidden
        WHMCS.UI.notify.error('Access denied');
    } else if (xhr.status === 500) {
        // Server error
        WHMCS.UI.notify.error('Server error occurred');
    }
});
```

### Retry Logic

```javascript
function ajaxWithRetry(options, maxRetries) {
    maxRetries = maxRetries || 3;
    
    return new Promise(function(resolve, reject) {
        function attempt(retryCount) {
            $.ajax(options)
                .done(resolve)
                .fail(function(xhr, status, error) {
                    if (retryCount < maxRetries) {
                        setTimeout(function() {
                            attempt(retryCount + 1);
                        }, 1000 * retryCount);
                    } else {
                        reject(error);
                    }
                });
        }
        
        attempt(1);
    });
}
```

## Best Practices

1. **Always handle errors** - Show user-friendly messages
2. **Show loading states** - Indicate activity to users
3. **Debounce searches** - Avoid excessive requests
4. **Use CSRF tokens** - Prevent CSRF attacks
5. **Test offline scenarios** - Handle network failures

## See Also

- [JavaScript Hooks](../whmcs-javascript-hooks.md)
- [Template Hooks](../whmcs-template-hooks.md)
- [Cart Templates](../whmcs-cart-templates.md)