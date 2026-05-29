# WHMCS Analytics Integration Workflow

## Purpose
Integrate web analytics platforms with WHMCS for comprehensive tracking.

## Prerequisites
- WHMCS installation
- Analytics account (GA4, Matomo, etc.)
- Template access

## Step-by-Step Process

### Step 1: Create Analytics Hook File

**Create hooks/analytics.php:**
```php
<?php
/**
 * WHMCS Analytics Integration
 * Supports Google Analytics 4, Matomo, and custom analytics
 */

use WHMCS\Config\Setting;

function getAnalyticsConfig() {
    return [
        'ga4_measurement_id' => Setting::getValue('GA4MeasurementId'),
        'ga4_api_secret' => Setting::getValue('GA4ApiSecret'),
        'matomo_url' => Setting::getValue('MatomoUrl'),
        'matomo_site_id' => Setting::getValue('MatomoSiteId'),
        'facebook_pixel_id' => Setting::getValue('FacebookPixelId'),
        'hotjar_id' => Setting::getValue('HotjarId')
    ];
}
```

### Step 2: Google Analytics 4 Setup

```php
<?php
/**
 * Add Google Analytics 4 to WHMCS
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $config = getAnalyticsConfig();
    $analytics = '';
    
    if (!empty($config['ga4_measurement_id'])) {
        $analytics .= '<!-- Google Analytics 4 -->' . "\n";
        $analytics .= '<script async src="https://www.googletagmanager.com/gtag/js?id=' . 
                     htmlspecialchars($config['ga4_measurement_id']) . '"></script>' . "\n";
        $analytics .= '<script>' . "\n";
        $analytics .= 'window.dataLayer = window.dataLayer || [];' . "\n";
        $analytics .= 'function gtag(){dataLayer.push(arguments);}' . "\n";
        $analytics .= "gtag('js', new Date());" . "\n";
        $analytics .= "gtag('config', '" . htmlspecialchars($config['ga4_measurement_id']) . "', {" . "\n";
        $analytics .= "    'page_title': '" . htmlspecialchars($vars['pagetitle'] ?? 'WHMCS') . "'," . "\n";
        $analytics .= "    'page_location': window.location.href," . "\n";
        $analytics .= "    'user_id': '" . ($vars['loggedin'] ? htmlspecialchars($vars['client']['id'] ?? '') : '') . "'" . "\n";
        $analytics .= '});' . "\n";
        $analytics .= '</script>' . "\n";
    }
    
    return $analytics;
});
```

### Step 3: GA4 E-commerce Events

```php
<?php
/**
 * Track e-commerce events with GA4
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $config = getAnalyticsConfig();
    
    if (empty($config['ga4_measurement_id'])) return '';
    
    $event = [
        'event' => 'purchase',
        'ecommerce' => [
            'transaction_id' => $vars['orderId'],
            'affiliation' => getCompanyName(),
            'value' => $vars['amount'],
            'currency' => 'USD',
            'tax' => $vars['tax'] ?? 0,
            'shipping' => $vars['shipping'] ?? 0,
            'items' => [[
                'item_id' => $vars['productId'],
                'item_name' => $vars['productName'],
                'item_category' => 'Hosting',
                'price' => $vars['price'],
                'quantity' => 1
            ]]
        ]
    ];
    
    return '<script>gtag("event", "purchase", ' . json_encode($event['ecommerce']) . ');</script>';
});

add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $config = getAnalyticsConfig();
    $events = '';
    
    if (empty($config['ga4_measurement_id'])) return '';
    
    $filename = $vars['filename'] ?? '';
    
    // View item event (product page)
    if ($filename === 'cart' && isset($vars['productinfo'])) {
        $product = $vars['productinfo'];
        $events .= '<script>' . "\n";
        $events .= "gtag('event', 'view_item', {" . "\n";
        $events .= "    items: [{" . "\n";
        $events .= "        item_id: '" . $product['id'] . "'," . "\n";
        $events .= "        item_name: '" . addslashes($product['name']) . "'," . "\n";
        $events .= "        item_category: 'Hosting'," . "\n";
        $events .= "        price: '" . ($product['pricing']['monthly']['price'] ?? 0) . "'," . "\n";
        $events .= "        quantity: 1" . "\n";
        $events .= "    }]" . "\n";
        $events .= "});" . "\n";
        $events .= '</script>' . "\n";
    }
    
    // View cart event
    if ($filename === 'cart' && isset($vars['cartitems'])) {
        $items = [];
        foreach ($vars['cartitems'] as $item) {
            $items[] = [
                'item_id' => $item['id'],
                'item_name' => $item['name'],
                'price' => $item['price'],
                'quantity' => 1
            ];
        }
        
        $total = array_sum(array_column($items, 'price'));
        
        $events .= '<script>' . "\n";
        $events .= "gtag('event', 'view_cart', {" . "\n";
        $events .= "    currency: 'USD'," . "\n";
        $events .= "    value: " . $total . "," . "\n";
        $events .= "    items: " . json_encode($items) . "\n";
        $events .= "});" . "\n";
        $events .= '</script>' . "\n";
    }
    
    return ['analytics_events' => $events];
});
```

### Step 4: GA4 Custom Events Tracking

```php
<?php
/**
 * Track custom events for WHMCS actions
 */
add_hook('TicketOpen', 1, function($vars) {
    $config = getAnalyticsConfig();
    
    if (empty($config['ga4_measurement_id'])) return '';
    
    return '<script>
        gtag("event", "ticket_opened", {
            ticket_department: "' . addslashes($vars['department']) . '",
            ticket_priority: "' . addslashes($vars['priority']) . '"
        });
    </script>';
});

add_hook('InvoicePaid', 1, function($vars) {
    $config = getAnalyticsConfig();
    
    if (empty($config['ga4_measurement_id'])) return '';
    
    return '<script>
        gtag("event", "invoice_paid", {
            invoice_id: "' . $vars['invoice_id'] . '",
            invoice_total: "' . $vars['total'] . '",
            payment_method: "' . addslashes($vars['payment_method']) . '"
        });
    </script>';
});

add_hook('DomainTransferComplete', 1, function($vars) {
    $config = getAnalyticsConfig();
    
    if (empty($config['ga4_measurement_id'])) return '';
    
    return '<script>
        gtag("event", "domain_transfer", {
            domain_name: "' . addslashes($vars['domain']) . '",
            transfer_status: "completed"
        });
    </script>';
});
```

### Step 5: Matomo Analytics Setup

```php
<?php
/**
 * Add Matomo Analytics to WHMCS
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $config = getAnalyticsConfig();
    $analytics = '';
    
    if (!empty($config['matomo_url']) && !empty($config['matomo_site_id'])) {
        $matomoUrl = rtrim($config['matomo_url'], '/');
        $siteId = htmlspecialchars($config['matomo_site_id']);
        
        $analytics .= '<!-- Matomo Analytics -->' . "\n";
        $analytics .= '<script>' . "\n";
        $analytics .= 'var _paq = window._paq = window._paq || [];' . "\n";
        $analytics .= '_paq.push(["trackPageView"]);' . "\n";
        $analytics .= '_paq.push(["enableLinkTracking"]);' . "\n";
        
        // Track user ID if logged in
        if ($vars['loggedin'] && isset($vars['client']['id'])) {
            $analytics .= '_paq.push(["setUserId", "' . $vars['client']['id'] . '"]);' . "\n";
        }
        
        $analytics .= '(function() {' . "\n";
        $analytics .= '    var u="' . $matomoUrl . '/";' . "\n";
        $analytics .= '    _paq.push(["setTrackerUrl", u+"matomo.php"]);' . "\n";
        $analytics .= '    _paq.push(["setSiteId", "' . $siteId . '"]);' . "\n";
        $analytics .= '    var d=document, g=d.createElement("script"), s=d.getElementsByTagName("script")[0];' . "\n";
        $analytics .= '    g.async=true; g.src=u+"matomo.js"; s.parentNode.insertBefore(g,s);' . "\n";
        $analytics .= '})();' . "\n";
        $analytics .= '</script>' . "\n";
    }
    
    return $analytics;
});
```

### Step 6: Facebook Pixel Setup

```php
<?php
/**
 * Add Facebook Pixel to WHMCS
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $config = getAnalyticsConfig();
    $pixel = '';
    
    if (!empty($config['facebook_pixel_id'])) {
        $pixelId = htmlspecialchars($config['facebook_pixel_id']);
        
        $pixel .= '<!-- Facebook Pixel -->' . "\n";
        $pixel .= '<script>' . "\n";
        $pixel .= '!function(f,b,e,v,n,t,s)' . "\n";
        $pixel .= '{if(f.fbq)return;n=f.fbq=function(){n.callMethod?' . "\n";
        $pixel .= 'n.callMethod.apply(n,arguments):n.queue.push(arguments)};' . "\n";
        $pixel .= 'if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version="2.0";' . "\n";
        $pixel .= 'n.queue=[];t=b.createElement(e);t.async=!0;' . "\n";
        $pixel .= 't.src=v;s=b.getElementsByTagName(e)[0];' . "\n";
        $pixel .= 's.parentNode.insertBefore(t,s)}(window, document,"script",' . "\n";
        $pixel .= '"https://connect.facebook.net/en_US/fbevents.js");' . "\n";
        $pixel .= 'fbq("init", "' . $pixelId . '");' . "\n";
        $pixel .= 'fbq("track", "PageView");' . "\n";
        
        // Track user data if logged in
        if ($vars['loggedin'] && isset($vars['client'])) {
            $pixel .= 'fbq("track", "PageView");' . "\n";
        }
        
        $pixel .= '</script>' . "\n";
        $pixel .= '<noscript><img height="1" width="1" style="display:none"' . "\n";
        $pixel .= 'src="https://www.facebook.com/tr?id=' . $pixelId . '&ev=PageView&noscript=1"/></noscript>' . "\n";
    }
    
    return $pixel;
});
```

### Step 7: Facebook Pixel Events

```php
<?php
/**
 * Track Facebook Pixel events
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $config = getAnalyticsConfig();
    
    if (empty($config['facebook_pixel_id'])) return '';
    
    return '<script>
        fbq("track", "Purchase", {
            content_ids: ["' . $vars['productId'] . '"],
            content_name: "' . addslashes($vars['productName']) . '",
            content_type: "product",
            value: ' . $vars['amount'] . ',
            currency: "USD"
        });
    </script>';
});

add_hook('TicketOpen', 1, function($vars) {
    $config = getAnalyticsConfig();
    
    if (empty($config['facebook_pixel_id'])) return '';
    
    return '<script>
        fbq("track", "Contact", {
            content_category: "' . addslashes($vars['department']) . '"
        });
    </script>';
});
```

### Step 8: Hotjar Integration

```php
<?php
/**
 * Add Hotjar to WHMCS
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $config = getAnalyticsConfig();
    $hotjar = '';
    
    if (!empty($config['hotjar_id'])) {
        $hotjarId = htmlspecialchars($config['hotjar_id']);
        
        $hotjar .= '<!-- Hotjar Tracking Code -->' . "\n";
        $hotjar .= '<script>' . "\n";
        $hotjar .= '(function(h,o,t,j,a,r){' . "\n";
        $hotjar .= '    h.hj=h.hj||function(){(h.hj.q=h.hj.q||[]).push(arguments)};' . "\n";
        $hotjar .= '    h._hjSettings={hjid:' . $hotjarId . ',hjsrc:"https://static.hotjar.com/c/hotjar-",' . "\n";
        $hotjar .= '    hjsv:6};' . "\n";
        $hotjar .= '    a=o.getElementsByTagName("head")[0];' . "\n";
        $hotjar .= '    r=o.createElement("script");r.async=1;' . "\n";
        $hotjar .= '    r.src=t+h._hjSettings.hjid+j+h._hjSettings.hjsv;' . "\n";
        $hotjar .= '    a.appendChild(r);' . "\n";
        $hotjar .= '})(window,document,"https://static.hotjar.com/c/hotjar-",".js?sv=");' . "\n";
        $hotjar .= '</script>' . "\n";
    }
    
    return $hotjar;
});
```

### Step 9: Client-Side Event Helper

```javascript
/**
 * WHMCS Analytics Helper Functions
 * Add to custom.js
 */

// Track page view with custom data
function trackPageView(pageTitle, pagePath) {
    // GA4
    if (typeof gtag !== 'undefined') {
        gtag('event', 'page_view', {
            page_title: pageTitle,
            page_location: pagePath
        });
    }
    
    // Matomo
    if (typeof _paq !== 'undefined') {
        _paq.push(['setCustomUrl', pagePath]);
        _paq.push(['trackPageView']);
    }
    
    // Facebook
    if (typeof fbq !== 'undefined') {
        fbq('track', 'PageView');
    }
}

// Track button clicks
function trackButtonClick(buttonName, buttonLocation) {
    if (typeof gtag !== 'undefined') {
        gtag('event', 'button_click', {
            button_name: buttonName,
            button_location: buttonLocation
        });
    }
}

// Track form submissions
function trackFormSubmit(formId, formName) {
    if (typeof gtag !== 'undefined') {
        gtag('event', 'form_submit', {
            form_id: formId,
            form_name: formName
        });
    }
}

// Track search queries
function trackSearch(searchTerm) {
    if (typeof gtag !== 'undefined') {
        gtag('event', 'search', {
            search_term: searchTerm
        });
    }
}

// Track video engagement
function trackVideoPlay(videoId, videoName) {
    if (typeof gtag !== 'undefined') {
        gtag('event', 'video_play', {
            video_id: videoId,
            video_name: videoName
        });
    }
}

// Initialize event listeners
document.addEventListener('DOMContentLoaded', function() {
    // Track all form submissions
    document.querySelectorAll('form').forEach(function(form) {
        form.addEventListener('submit', function() {
            trackFormSubmit(form.id || 'unknown', form.name || form.action);
        });
    });
    
    // Track outbound links
    document.querySelectorAll('a[href^="http"]').forEach(function(link) {
        if (!link.href.includes(window.location.hostname)) {
            link.addEventListener('click', function() {
                if (typeof gtag !== 'undefined') {
                    gtag('event', 'outbound_click', {
                        link_url: link.href,
                        link_text: link.textContent
                    });
                }
            });
        }
    });
});
```

### Step 10: Analytics Consent Management

```php
<?php
/**
 * Cookie consent for analytics
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $analytics = '';
    
    // Load scripts only after consent
    $analytics .= '<script>' . "\n";
    $analytics .= 'window.addEventListener("consentGranted", function() {' . "\n";
    $analytics .= '    // Load analytics scripts here' . "\n";
    $analytics .= '    loadAnalyticsScripts();' . "\n";
    $analytics .= '});' . "\n";
    $analytics .= '' . "\n";
    $analytics .= 'function loadAnalyticsScripts() {' . "\n";
    $analytics .= '    // Load GA4' . "\n";
    $analytics .= '    ' . "\n";
    $analytics .= '    // Load Matomo' . "\n";
    $analytics .= '    ' . "\n";
    $analytics .= '    // Load Facebook Pixel' . "\n";
    $analytics .= '    ' . "\n";
    $analytics .= '}' . "\n";
    $analytics .= '</script>' . "\n";
    
    return $analytics;
});
```

## Best Practices
- Use consent management for GDPR compliance
- Track only necessary metrics
- Anonymize IP addresses when required
- Set up goals and conversions
- Monitor data quality regularly
- Use server-side tracking for sensitive data
- Implement cross-domain tracking if needed
- Test analytics implementation thoroughly
- Document tracking setup
- Review analytics reports regularly
