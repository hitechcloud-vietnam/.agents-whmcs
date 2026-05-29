# WHMCS Tracking Pixel Configuration Workflow

## Purpose
Configure and manage tracking pixels for marketing attribution and conversion tracking.

## Prerequisites
- WHMCS installation
- Template access
- Marketing platform accounts
- GDPR compliance considerations

## Step-by-Step Process

### Step 1: Create Tracking Pixel Hook File

**Create hooks/tracking_pixels.php:**
```php
<?php
/**
 * WHMCS Tracking Pixel Manager
 * Centralized tracking pixel configuration
 */

use WHMCS\Config\Setting;

class TrackingPixelManager {
    private $pixels = [];
    
    public function __construct() {
        $this->loadPixels();
    }
    
    private function loadPixels() {
        // Load tracking pixel configurations from WHMCS settings
        $this->pixels = [
            'facebook' => [
                'enabled' => (bool)Setting::getValue('FacebookPixelEnabled'),
                'pixel_id' => Setting::getValue('FacebookPixelId'),
                'events' => $this->getFacebookEvents()
            ],
            'google' => [
                'enabled' => (bool)Setting::getValue('GoogleTagEnabled'),
                'tag_id' => Setting::getValue('GoogleTagId'),
                'conversions' => $this->getGoogleConversions()
            ],
            'twitter' => [
                'enabled' => (bool)Setting::getValue('TwitterPixelEnabled'),
                'pixel_id' => Setting::getValue('TwitterPixelId')
            ],
            'linkedin' => [
                'enabled' => (bool)Setting::getValue('LinkedInInsightEnabled'),
                'partner_id' => Setting::getValue('LinkedInPartnerId')
            ],
            'tiktok' => [
                'enabled' => (bool)Setting::getValue('TikTokPixelEnabled'),
                'pixel_id' => Setting::getValue('TikTokPixelId')
            ],
            'pinterest' => [
                'enabled' => (bool)Setting::getValue('PinterestTagEnabled'),
                'tag_id' => Setting::getValue('PinterestTagId')
            ],
            'snapchat' => [
                'enabled' => (bool)Setting::getValue('SnapchatPixelEnabled'),
                'pixel_id' => Setting::getValue('SnapchatPixelId')
            ]
        ];
    }
    
    private function getFacebookEvents() {
        return [
            'PageView' => true,
            'ViewContent' => true,
            'AddToCart' => true,
            'InitiateCheckout' => true,
            'AddPaymentInfo' => true,
            'Purchase' => true,
            'Lead' => true,
            'CompleteRegistration' => true
        ];
    }
    
    private function getGoogleConversions() {
        return [
            'purchase' => Setting::getValue('GoogleConversionId'),
            'lead' => Setting::getValue('GoogleConversionIdLead')
        ];
    }
    
    public function getPixels() {
        return $this->pixels;
    }
    
    public function isEnabled($platform) {
        return isset($this->pixels[$platform]) && 
               $this->pixels[$platform]['enabled'] &&
               !empty($this->pixels[$platform]['pixel_id'] ?? $this->pixels[$platform]['partner_id']);
    }
}

$trackingManager = new TrackingPixelManager();
```

### Step 2: Base Tracking Code

```php
<?php
/**
 * Add base tracking codes to head
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $manager = new TrackingPixelManager();
    $scripts = '';
    
    // Facebook Pixel Base Code
    if ($manager->isEnabled('facebook')) {
        $pixelId = $manager->pixels['facebook']['pixel_id'];
        $scripts .= '<!-- Facebook Pixel Code -->' . "\n";
        $scripts .= '<script>' . "\n";
        $scripts .= '!function(f,b,e,v,n,t,s){if(f.fbq)return;n=f.fbq=function(){n.callMethod?' . "\n";
        $scripts .= 'n.callMethod.apply(n,arguments):n.queue.push(arguments)};if(!f._fbq)f._fbq=n;' . "\n";
        $scripts .= 'n.push=n;n.loaded=!0;n.version="2.0";n.queue=[];t=b.createElement(e);t.async=!0;' . "\n";
        $scripts .= 't.src=v;s=b.getElementsByTagName(e)[0];s.parentNode.insertBefore(t,s)}(window,' . "\n";
        $scripts .= 'document,"script","https://connect.facebook.net/en_US/fbevents.js");' . "\n";
        $scripts .= 'fbq("init", "' . $pixelId . '");' . "\n";
        $scripts .= 'fbq("track", "PageView");' . "\n";
        $scripts .= '</script>' . "\n";
        $scripts .= '<noscript><img height="1" width="1" style="display:none"' . "\n";
        $scripts .= 'src="https://www.facebook.com/tr?id=' . $pixelId . '&ev=PageView&noscript=1"/></noscript>' . "\n";
    }
    
    // Twitter Pixel Base Code
    if ($manager->isEnabled('twitter')) {
        $pixelId = $manager->pixels['twitter']['pixel_id'];
        $scripts .= '<!-- Twitter Universal Website Tag -->' . "\n";
        $scripts .= '<script>' . "\n";
        $scripts .= '!function(e,t,n,s,u,a){e.twq||(a=e.twq=function(){a.exe?a.exe.apply(a,arguments):' . "\n";
        $scripts .= 'a.queue.push(arguments);},a.version="1.1",a.queue=[],u=t.createElement(n),' . "\n";
        $scripts .= 'u.async=!0,u.src=s,a=t.getElementsByTagName(n)[0],a.parentNode.insertBefore(u,a))}' . "\n";
        $scripts .= '(window,document,"script","https://static.ads-twitter.com/uwtf.js");' . "\n";
        $scripts .= 'twq("init","' . $pixelId . '");twq("track","PageView");' . "\n";
        $scripts .= '</script>' . "\n";
    }
    
    // LinkedIn Insight Tag
    if ($manager->isEnabled('linkedin')) {
        $partnerId = $manager->pixels['linkedin']['partner_id'];
        $scripts .= '<!-- LinkedIn Insight Tag -->' . "\n";
        $scripts .= '<script type="text/javascript">' . "\n";
        $scripts .= '!function(l){if(!l){var l=window;l._linkedin_data_partner_ids=l._linkedin_data_partner_ids||[];' . "\n";
        $scripts .= 'l._linkedin_data_partner_ids.push("' . $partnerId . '");' . "\n";
        $scripts .= '}}(window);' . "\n";
        $scripts .= '</script>' . "\n";
        $scripts .= '<script async src="https://snap.licdn.com/li.lms-analytics/insight.min.js"></script>' . "\n";
        $scripts .= '<noscript><img height="1" width="1" style="display:none;" alt="" ' . "\n";
        $scripts .= 'src="https://dc.ads.linkedin.com/collect/?pid=' . $partnerId . '&fmt=gif"/></noscript>' . "\n";
    }
    
    // TikTok Pixel
    if ($manager->isEnabled('tiktok')) {
        $pixelId = $manager->pixels['tiktok']['pixel_id'];
        $scripts .= '<!-- TikTok Pixel -->' . "\n";
        $scripts .= '<script>' . "\n";
        $scripts .= '!function (w, d, t) {' . "\n";
        $scripts .= 'w.TikTokPlugin = function (c) { for (var o = t.split("."), i = o.shift(), e = w, n = 0; n < o.length; n++)e = e[o[n]] = e[o[n]] || {}; ' . "\n";
        $scripts .= 'e[c] = e[c] || {}; e[c].call = function () { e[c].call ? e[c].call.apply(this, arguments) : e[c].queue.push(arguments); }; ' . "\n";
        $scripts .= 'e[c].queue = []; }( ["setPixelCode", "track"] );' . "\n";
        $scripts .= 'var sdk = document.createElement("script"); sdk.async = 1; sdk.src = "https://analytics.tiktok.com/i18n/pixel/events.js"; ' . "\n";
        $scripts .= 'document.head.appendChild(sdk); ' . "\n";
        $scripts .= 'window.tiktokPixel = window.tiktokPixel || {}; window.tiktokPixel.methods = ["setPixelCode", "track", "trackLink"]; ' . "\n";
        $scripts .= 'window.tiktokPixel.setPixelCode = function (id) { window.tiktokPixel.pixelId = id; }; ' . "\n";
        $scripts .= 'window.tiktokPixel.setPixelCode("' . $pixelId . '"); ' . "\n";
        $scripts .= '</script>' . "\n";
    }
    
    return $scripts;
});
```

### Step 3: Conversion Events

```php
<?php
/**
 * Track conversion events
 */

// Purchase/Order Complete
add_hook('AfterModuleCreate', 1, function($vars) {
    $manager = new TrackingPixelManager();
    $scripts = '';
    
    // Facebook Purchase Event
    if ($manager->isEnabled('facebook')) {
        $scripts .= '<script>' . "\n";
        $scripts .= 'fbq("track", "Purchase", {' . "\n";
        $scripts .= '    content_ids: ["' . $vars['productId'] . '"],' . "\n";
        $scripts .= '    content_name: "' . addslashes($vars['productName']) . '",' . "\n";
        $scripts .= '    content_type: "product",' . "\n";
        $scripts .= '    value: ' . number_format($vars['amount'], 2, '.', '') . ',' . "\n";
        $scripts .= '    currency: "USD"' . "\n";
        $scripts .= '});' . "\n";
        $scripts .= '</script>' . "\n";
    }
    
    // Twitter Purchase Event
    if ($manager->isEnabled('twitter')) {
        $scripts .= '<script>' . "\n";
        $scripts .= 'twq("track","Purchase",{' . "\n";
        $scripts .= '    value: ' . number_format($vars['amount'], 2, '.', '') . ',' . "\n";
        $scripts .= '    currency: "USD",' . "\n";
        $scripts .= '    transaction_id: "' . $vars['orderId'] . '"' . "\n";
        $scripts .= '});' . "\n";
        $scripts .= '</script>' . "\n";
    }
    
    // TikTok Purchase Event
    if ($manager->isEnabled('tiktok')) {
        $scripts .= '<script>' . "\n";
        $scripts .= 'tiktokPixel.track("Purchase", {' . "\n";
        $scripts .= '    content_id: "' . $vars['productId'] . '",' . "\n";
        $scripts .= '    content_type: "product",' . "\n";
        $scripts .= '    value: ' . number_format($vars['amount'], 2, '.', '') . ',' . "\n";
        $scripts .= '    currency: "USD"' . "\n";
        $scripts .= '});' . "\n";
        $scripts .= '</script>' . "\n";
    }
    
    return $scripts;
});

// Lead Generation (Ticket Opened)
add_hook('TicketOpen', 1, function($vars) {
    $manager = new TrackingPixelManager();
    $scripts = '';
    
    if ($manager->isEnabled('facebook')) {
        $scripts .= '<script>' . "\n";
        $scripts .= 'fbq("track", "Lead", {' . "\n";
        $scripts .= '    content_category: "' . addslashes($vars['department']) . '",' . "\n";
        $scripts .= '    content_name: "Support Ticket"' . "\n";
        $scripts .= '});' . "\n";
        $scripts .= '</script>' . "\n";
    }
    
    return $scripts;
});

// Complete Registration
add_hook('ClientAdd', 1, function($vars) {
    $manager = new TrackingPixelManager();
    $scripts = '';
    
    if ($manager->isEnabled('facebook')) {
        $scripts .= '<script>' . "\n";
        $scripts .= 'fbq("track", "CompleteRegistration", {' . "\n";
        $scripts .= '    content_name: "Client Registration"' . "\n";
        $scripts .= '});' . "\n";
        $scripts .= '</script>' . "\n";
    }
    
    return $scripts;
});
```

### Step 4: Page-Specific Events

```php
<?php
/**
 * Track page-specific events
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $manager = new TrackingPixelManager();
    $scripts = '';
    
    $filename = $vars['filename'] ?? '';
    
    // View Content (Product Page)
    if ($filename === 'cart' && isset($vars['productinfo'])) {
        $product = $vars['productinfo'];
        
        if ($manager->isEnabled('facebook')) {
            $scripts .= '<script>' . "\n";
            $scripts .= 'fbq("track", "ViewContent", {' . "\n";
            $scripts .= '    content_ids: ["' . $product['id'] . '"],' . "\n";
            $scripts .= '    content_name: "' . addslashes($product['name']) . '",' . "\n";
            $scripts .= '    content_type: "product",' . "\n";
            $scripts .= '    value: ' . number_format($product['pricing']['monthly']['price'] ?? 0, 2, '.', '') . ',' . "\n";
            $scripts .= '    currency: "USD"' . "\n";
            $scripts .= '});' . "\n";
            $scripts .= '</script>' . "\n";
        }
    }
    
    // Add to Cart
    if ($filename === 'cart' && isset($_GET['a']) && $_GET['a'] === 'add') {
        if ($manager->isEnabled('facebook')) {
            $scripts .= '<script>' . "\n";
            $scripts .= 'fbq("track", "AddToCart", {' . "\n";
            $scripts .= '    content_ids: ["' . ($vars['productinfo']['id'] ?? '') . '"],' . "\n";
            $scripts .= '    content_name: "' . addslashes($vars['productinfo']['name'] ?? '') . '",' . "\n";
            $scripts .= '    content_type: "product",' . "\n";
            $scripts .= '    value: ' . number_format($vars['productinfo']['pricing']['monthly']['price'] ?? 0, 2, '.', '') . ',' . "\n";
            $scripts .= '    currency: "USD"' . "\n";
            $scripts .= '});' . "\n";
            $scripts .= '</script>' . "\n";
        }
    }
    
    // Initiate Checkout
    if ($filename === 'cart' && isset($vars['cartitems'])) {
        if ($manager->isEnabled('facebook')) {
            $itemIds = array_column($vars['cartitems'], 'id');
            $total = array_sum(array_column($vars['cartitems'], 'price'));
            
            $scripts .= '<script>' . "\n";
            $scripts .= 'fbq("track", "InitiateCheckout", {' . "\n";
            $scripts .= '    num_items: ' . count($itemIds) . ',' . "\n";
            $scripts .= '    value: ' . number_format($total, 2, '.', '') . ',' . "\n";
            $scripts .= '    currency: "USD"' . "\n";
            $scripts .= '});' . "\n";
            $scripts .= '</script>' . "\n";
        }
    }
    
    // Add Payment Info
    if ($filename === 'checkout' && isset($_POST['payment_method'])) {
        if ($manager->isEnabled('facebook')) {
            $scripts .= '<script>' . "\n";
            $scripts .= 'fbq("track", "AddPaymentInfo");' . "\n";
            $scripts .= '</script>' . "\n";
        }
    }
    
    // Domain Search
    if ($filename === 'domainchecker') {
        if ($manager->isEnabled('facebook')) {
            $scripts .= '<script>' . "\n";
            $scripts .= 'fbq("track", "Search", {' . "\n";
            $scripts .= '    search_string: "' . addslashes($_GET['domain'] ?? '') . '"' . "\n";
            $scripts .= '});' . "\n";
            $scripts .= '</script>' . "\n";
        }
    }
    
    return ['tracking_events' => $scripts];
});
```

### Step 5: Pinterest Tag

```php
<?php
/**
 * Pinterest Tag implementation
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $manager = new TrackingPixelManager();
    $pinterest = '';
    
    if ($manager->isEnabled('pinterest')) {
        $tagId = $manager->pixels['pinterest']['tag_id'];
        
        $pinterest .= '<!-- Pinterest Tag -->' . "\n";
        $pinterest .= '<script>' . "\n";
        $pinterest .= '!function(e){if(!window.pintrk){window.pintrk=function(){window.pintrk.queue.push(' . "\n";
        $pinterest .= 'Array.prototype.slice.call(arguments))};var' . "\n";
        $pinterest .= 'n=window.pintrk;n.queue=[],n.version="3.0";var' . "\n";
        $pinterest .= 't=document.createElement("script");t.async=!0,t.src=e;var' . "\n";
        $pinterest .= 'r=document.getElementsByTagName("script")[0];r.parentNode.insertBefore(t,r)}}(' . "\n";
        $pinterest .= '"https://ct.pinterest.com/v3/?tid=' . $tagId . '&event=init");' . "\n";
        $pinterest .= 'pintrk("load", "' . $tagId . '");' . "\n";
        $pinterest .= 'pintrk("page");' . "\n";
        $pinterest .= '</script>' . "\n";
        $pinterest .= '<noscript>' . "\n";
        $pinterest .= '<img height="1" width="1" style="display:none;" alt="" ' . "\n";
        $pinterest .= 'src="https://ct.pinterest.com/v3/?tid=' . $tagId . '&noscript=1"/>' . "\n";
        $pinterest .= '</noscript>' . "\n";
    }
    
    return $pinterest;
});
```

### Step 6: Snapchat Pixel

```php
<?php
/**
 * Snapchat Pixel implementation
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $manager = new TrackingPixelManager();
    $snapchat = '';
    
    if ($manager->isEnabled('snapchat')) {
        $pixelId = $manager->pixels['snapchat']['pixel_id'];
        
        $snapchat .= '<!-- Snapchat Pixel -->' . "\n";
        $snapchat .= '<script>' . "\n";
        $snapchat .= '!function(e,a,n,t){if(a.getElementById("snapxpixel"))return;' . "\n";
        $snapchat .= 'e[n]=function(){e[n].q=e[n].q||[];e[n].q.push(arguments)},e[n].t=1*new Date,' . "\n";
        $snapchat .= 'e[n].c=t;var c=a.createElement("script");c.async=!0,c.id="snapxpixel",' . "\n";
        $snapchat .= 'c.src="https://sc-static.net/scevent.min.js";var r=a.getElementsByTagName("script")[0];' . "\n";
        $snapchat .= 'r.parentNode.insertBefore(c,r)}(window,document,"snapxpixel","' . $pixelId . '");' . "\n";
        $snapchat .= 'snapxpixel("track", "PAGE_VIEW");' . "\n";
        $snapchat .= '</script>' . "\n";
    }
    
    return $snapchat;
});
```

### Step 7: Custom Event Helper

```javascript
/**
 * WHMCS Tracking Pixel Helper
 * Add to custom.js
 */

// Generic pixel event tracking
function trackPixelEvent(platform, eventName, eventData) {
    // Facebook
    if (typeof fbq !== 'undefined') {
        fbq('track', eventName, eventData);
    }
    
    // Twitter
    if (typeof twq !== 'undefined') {
        twq('track', eventName, eventData);
    }
    
    // TikTok
    if (typeof tiktokPixel !== 'undefined') {
        tiktokPixel.track(eventName, eventData);
    }
    
    // Pinterest
    if (typeof pintrk !== 'undefined') {
        pintrk('track', eventName, eventData);
    }
    
    // Snapchat
    if (typeof snapxpixel !== 'undefined') {
        snapxpixel('track', eventName, eventData);
    }
}

// Usage examples
document.addEventListener('DOMContentLoaded', function() {
    // Track custom button clicks
    document.querySelectorAll('[data-track-click]').forEach(function(element) {
        element.addEventListener('click', function() {
            var eventName = this.getAttribute('data-track-click');
            trackPixelEvent('facebook', eventName, {
                content_name: this.textContent
            });
        });
    });
    
    // Track email signup forms
    document.querySelectorAll('form[data-track-signup]').forEach(function(form) {
        form.addEventListener('submit', function() {
            trackPixelEvent('facebook', 'CompleteRegistration', {});
        });
    });
    
    // Track contact form submissions
    document.querySelectorAll('form[data-track-contact]').forEach(function(form) {
        form.addEventListener('submit', function() {
            trackPixelEvent('facebook', 'Contact', {
                content_category: 'Support'
            });
        });
    });
});
```

### Step 8: Consent Management

```php
<?php
/**
 * Cookie consent integration for tracking pixels
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $scripts = '';
    
    // Only load tracking pixels after consent
    $scripts .= '<script>' . "\n";
    $scripts .= 'function loadTrackingPixels() {' . "\n";
    $scripts .= '    // Check if user has given consent' . "\n";
    $scripts .= '    const consent = localStorage.getItem("marketing_consent");' . "\n";
    $scripts .= '    if (consent !== "granted") return;' . "\n";
    $scripts .= '' . "\n";
    $scripts .= '    // Initialize tracking pixels here' . "\n";
    $scripts .= '    // Facebook, Twitter, TikTok, etc.' . "\n";
    $scripts .= '}' . "\n";
    $scripts .= '' . "\n";
    $scripts .= '// Load pixels when consent is granted' . "\n";
    $scripts .= 'window.addEventListener("marketingConsentGranted", loadTrackingPixels);' . "\n";
    $scripts .= '' . "\n";
    $scripts .= '// Check consent on page load' . "\n";
    $scripts .= 'document.addEventListener("DOMContentLoaded", function() {' . "\n";
    $scripts .= '    if (localStorage.getItem("marketing_consent") === "granted") {' . "\n";
    $scripts .= '        loadTrackingPixels();' . "\n";
    $scripts .= '    }' . "\n";
    $scripts .= '});' . "\n";
    $scripts .= '</script>' . "\n";
    
    return $scripts;
});
```

## Best Practices
- Implement consent management for GDPR compliance
- Use standard event names for consistency
- Track only necessary conversion events
- Implement server-side tracking for sensitive data
- Test tracking across all devices
- Monitor data quality and accuracy
- Update tracking codes regularly
- Document all tracking implementations
- Consider using tag management system
- Review attribution models periodically
