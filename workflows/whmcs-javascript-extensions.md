# WHMCS JavaScript Extensions Workflow

## Purpose
Guide developers through adding custom JavaScript functionality to WHMCS.

## Prerequisites
- WHMCS installation
- JavaScript/jQuery knowledge
- Understanding of WHMCS hooks system
- Browser developer tools

## Steps

### Phase 1: JavaScript Architecture

1. Locate JavaScript files
   ```
   /whmcs/templates/six/js/
   ├── bootstrap.min.js
   ├── fontawesome.min.js
   ├── jquery.min.js
   ├── main.js
   └── custom.js
   ```

2. WHMCS JavaScript dependencies
   - jQuery (v3.x in WHMCS 8.x)
   - Bootstrap 4 JS components
   - Font Awesome icons
   - WHMCS native AJAX functions

3. Include order
   - jQuery loads first
   - Bootstrap JS
   - WHMCS core JS
   - Custom JS files

### Phase 2: Adding Custom JavaScript

#### Method 1: Hook-based JavaScript
1. Create PHP hook file
   ```php
   // hooks/clientarea_page_head.php
   <?php
   add_hook('ClientAreaPageHead', 1, function($vars) {
       echo '<script>
           document.addEventListener("DOMContentLoaded", function() {
               console.log("WHMCS Custom JS Loaded");
           });
       </script>';
   });
   ```

2. Add JavaScript file via hook
   ```php
   add_hook('ClientAreaPageHead', 1, function($vars) {
       echo '<script src="' . $vars['WEB_ROOT'] 
           . '/templates/yourtheme/js/custom.js" defer></script>';
   });
   ```

#### Method 2: Template Direct JavaScript
```smarty
{* In your template file *}
<script>
(function($) {
    'use strict';
    
    $(document).ready(function() {
        initCustomFeature();
    });
    
    function initCustomFeature() {
        // Your custom code
    }
})(jQuery);
</script>
```

### Phase 3: Common JavaScript Customizations

#### Form Validation Enhancement
```javascript
// Custom form validation
(function($) {
    'use strict';
    
    $(document).ready(function() {
        // Add custom validation to forms
        $('form[data-validate="custom"]').on('submit', function(e) {
            var isValid = true;
            
            $(this).find('[required]').each(function() {
                if (!$(this).val().trim()) {
                    isValid = false;
                    $(this).addClass('is-invalid');
                }
            });
            
            if (!isValid) {
                e.preventDefault();
                showValidationMessage();
            }
        });
    });
    
    function showValidationMessage() {
        $.notify({
            message: 'Please fill in all required fields',
            type: 'danger'
        });
    }
})(jQuery);
```

#### AJAX Data Loading
```javascript
// Custom AJAX loader
(function($) {
    'use strict';
    
    const CustomLoader = {
        loadProducts: function(categoryId) {
            $.ajax({
                url: 'ajax.php',
                type: 'POST',
                data: {
                    action: 'getProducts',
                    category: categoryId
                },
                beforeSend: function() {
                    $('#product-list').html('<div class="loader">Loading...</div>');
                },
                success: function(response) {
                    $('#product-list').html(response.html);
                },
                error: function(xhr, status, error) {
                    console.error('Error loading products:', error);
                }
            });
        },
        
        addToCart: function(productId, options) {
            return $.ajax({
                url: 'cart.php',
                type: 'POST',
                data: {
                    a: 'add',
                    product: productId,
                    options: options
                }
            });
        }
    };
    
    window.CustomLoader = CustomLoader;
})(jQuery);
```

#### Dynamic Content Updates
```javascript
// Live search functionality
(function($) {
    'use strict';
    
    $(document).ready(function() {
        var searchTimeout = null;
        
        $('#search-input').on('input', function() {
            var query = $(this).val();
            
            clearTimeout(searchTimeout);
            searchTimeout = setTimeout(function() {
                if (query.length >= 3) {
                    performSearch(query);
                }
            }, 300);
        });
        
        function performSearch(query) {
            $.post('search.php', { q: query }, function(response) {
                displayResults(response.results);
            });
        }
        
        function displayResults(results) {
            var html = results.map(function(item) {
                return '<div class="search-result">' +
                       '<a href="' + item.url + '">' + item.title + '</a>' +
                       '</div>';
            }).join('');
            
            $('#search-results').html(html).show();
        }
    });
})(jQuery);
```

#### Modal Interactions
```javascript
// Custom modal handling
(function($) {
    'use strict';
    
    $(document).ready(function() {
        // Custom modal trigger
        $(document).on('click', '[data-modal-trigger]', function(e) {
            e.preventDefault();
            var target = $(this).data('modal-trigger');
            showCustomModal(target);
        });
        
        function showCustomModal(modalId) {
            $('#' + modalId).modal({
                backdrop: 'static',
                keyboard: true
            });
        }
        
        // Modal form submission
        $(document).on('submit', '[data-ajax-form]', function(e) {
            e.preventDefault();
            var $form = $(this);
            var action = $form.attr('action');
            
            $.ajax({
                url: action,
                type: 'POST',
                data: $form.serialize(),
                success: function(response) {
                    if (response.success) {
                        closeModal($form.closest('.modal'));
                        showNotification(response.message, 'success');
                    }
                }
            });
        });
    });
})(jQuery);
```

### Phase 4: WHMCS API Integration with JavaScript

#### Using WHMCS AJAX API
```javascript
// WHMCS client area AJAX
(function($) {
    'use strict';
    
    const WHMCSApi = {
        endpoint: 'includes/api.php',
        
        call: function(params) {
            return $.ajax({
                url: this.endpoint,
                type: 'POST',
                dataType: 'json',
                data: $.extend({
                    // Required API params
                    // Add your API credentials
                }, params)
            });
        },
        
        getClient: function(clientId) {
            return this.call({
                action: 'GetClientsDetails',
                clientid: clientId
            });
        },
        
        createTicket: function(data) {
            return this.call({
                action: 'OpenTicket',
                subject: data.subject,
                message: data.message,
                priority: data.priority || 'Medium'
            });
        },
        
        getInvoices: function(clientId) {
            return this.call({
                action: 'getInvoices',
                clientid: clientId,
                limitnum: 10
            });
        }
    };
    
    window.WHMCSApi = WHMCSApi;
})(jQuery);
```

### Phase 5: Event Handling

#### Custom Event Listeners
```javascript
// WHMCS page event handling
(function($) {
    'use strict';
    
    // Page-specific initialization
    $(document).on('whmcs.page.load', function(event, pageInfo) {
        switch(pageInfo.page) {
            case 'cart':
                initCartPage();
                break;
            case 'clientarea':
                initClientAreaPage();
                break;
            case 'domainchecker':
                initDomainChecker();
                break;
        }
    });
    
    function initCartPage() {
        // Cart-specific JS
        $('body').addClass('cart-page-loaded');
    }
    
    function initClientAreaPage() {
        // Client area specific JS
    }
    
    function initDomainChecker() {
        // Domain checker specific JS
    }
    
    // Custom event triggers
    $(document).trigger('custom.ready');
    $(document).on('custom.submit', function(e, data) {
        processCustomData(data);
    });
})(jQuery);
```

### Phase 6: Error Handling

```javascript
// Global error handler
(function($) {
    'use strict';
    
    // AJAX error handling
    $(document).ajaxError(function(event, xhr, settings, thrownError) {
        console.error('AJAX Error:', {
            status: xhr.status,
            response: xhr.responseText,
            error: thrownError
        });
        
        if (xhr.status === 401) {
            // Redirect to login
            window.location.href = 'login.php';
        } else if (xhr.status === 500) {
            showSystemError();
        }
    });
    
    // Global JS error handler
    window.onerror = function(message, source, lineno, colno, error) {
        console.error('JS Error:', {
            message: message,
            source: source,
            line: lineno,
            column: colno,
            stack: error ? error.stack : null
        });
        
        // Send to error tracking service
        logError({
            message: message,
            source: source,
            lineno: lineno,
            colno: colno,
            userAgent: navigator.userAgent
        });
        
        return false;
    };
})(jQuery);
```

### Phase 7: Performance Optimization

#### Lazy Loading
```javascript
// Lazy load images and content
(function($) {
    'use strict';
    
    const LazyLoader = {
        init: function() {
            if ('IntersectionObserver' in window) {
                this.observeImages();
            } else {
                this.fallbackLoad();
            }
        },
        
        observeImages: function() {
            const observer = new IntersectionObserver(function(entries) {
                entries.forEach(function(entry) {
                    if (entry.isIntersecting) {
                        const img = entry.target;
                        img.src = img.dataset.src;
                        img.classList.add('loaded');
                        observer.unobserve(img);
                    }
                });
            });
            
            $('img[data-src]').each(function() {
                observer.observe(this);
            });
        }
    };
    
    $(document).ready(function() {
        LazyLoader.init();
    });
})(jQuery);
```

### Phase 8: Testing JavaScript

1. Console testing
   - Check for errors in browser console
   - Verify jQuery availability
   - Test AJAX calls

2. Browser compatibility
   - Test in Chrome, Firefox, Safari, Edge
   - Check ES6/ES7 compatibility
   - Verify polyfill requirements

3. Mobile testing
   - Touch events
   - Responsive behavior
   - Performance on mobile

## Related Workflows
- whmcs-template-modification
- whmcs-css-customization
- whmcs-api-integration
- whmcs-form-styling