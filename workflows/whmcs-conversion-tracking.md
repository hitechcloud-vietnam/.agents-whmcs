# WHMCS Conversion Tracking Workflow

## Purpose
Set up comprehensive conversion tracking for WHMCS to measure marketing effectiveness.

## Prerequisites
- WHMCS installation
- Analytics accounts
- Marketing platform access
- Template access

## Step-by-Step Process

### Step 1: Create Conversion Tracking Hook File

**Create hooks/conversion_tracking.php:**
```php
<?php
/**
 * WHMCS Conversion Tracking Manager
 */

use WHMCS\Database\Capsule;
use WHMCS\Config\Setting;

class ConversionTracking {
    
    public static function getConversionData($type, $params = []) {
        switch ($type) {
            case 'order':
                return self::getOrderConversionData($params);
            case 'signup':
                return self::getSignupConversionData($params);
            case 'ticket':
                return self::getTicketConversionData($params);
            case 'domain':
                return self::getDomainConversionData($params);
            default:
                return [];
        }
    }
    
    private static function getOrderConversionData($params) {
        $orderId = $params['order_id'] ?? 0;
        
        $order = Capsule::table('tblorders')
            ->where('id', $orderId)
            ->first();
        
        if (!$order) return [];
        
        $items = Capsule::table('tblorderitems')
            ->where('orderid', $orderId)
            ->get();
        
        return [
            'transaction_id' => 'ORD-' . $order->id,
            'affiliation' => Setting::getValue('CompanyName'),
            'value' => $order->totalamount,
            'tax' => $order->tax,
            'shipping' => 0,
            'currency' => $order->currency,
            'items' => $items->map(function($item) {
                return [
                    'sku' => $item->id,
                    'name' => $item->description,
                    'category' => 'Hosting',
                    'price' => $item->amount,
                    'quantity' => 1
                ];
            })->toArray()
        ];
    }
    
    private static function getSignupConversionData($params) {
        $userId = $params['user_id'] ?? 0;
        
        $client = Capsule::table('tblclients')
            ->where('id', $userId)
            ->first();
        
        if (!$client) return [];
        
        return [
            'transaction_id' => 'REG-' . $client->id,
            'currency' => 'USD',
            'new_customer' => true
        ];
    }
    
    private static function getTicketConversionData($params) {
        $ticketId = $params['ticket_id'] ?? 0;
        
        $ticket = Capsule::table('tbltickets')
            ->where('id', $ticketId)
            ->first();
        
        if (!$ticket) return [];
        
        return [
            'ticket_id' => 'TKT-' . $ticket->id,
            'department' => $ticket->did,
            'priority' => $ticket->urgency
        ];
    }
    
    private static function getDomainConversionData($params) {
        return [
            'domain_name' => $params['domain'] ?? '',
            'tld' => $params['tld'] ?? '',
            'registration_period' => $params['years'] ?? 1
        ];
    }
}
```

### Step 2: Order Completion Tracking

```php
<?php
/**
 * Track order completions
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $tracking = ConversionTracking::getConversionData('order', ['order_id' => $vars['orderId']]);
    
    $scripts = '';
    $scripts .= '<!-- Conversion Tracking - Order -->' . "\n";
    $scripts .= '<script>' . "\n";
    $scripts .= 'window.whmcsConversionData = ' . json_encode($tracking) . ';' . "\n";
    $scripts .= 'window.whmcsConversionType = "order";' . "\n";
    $scripts .= 'window.whmcsConversionValue = ' . $tracking['value'] . ';' . "\n";
    $scripts .= 'window.whmcsConversionCurrency = "' . $tracking['currency'] . '";' . "\n";
    $scripts .= 'document.dispatchEvent(new CustomEvent("whmcsConversion", {detail: window.whmcsConversionData}));' . "\n";
    $scripts .= '</script>' . "\n";
    
    return $scripts;
});
```

### Step 3: Signup/Registration Tracking

```php
<?php
/**
 * Track client registrations
 */
add_hook('ClientAdd', 1, function($vars) {
    $tracking = ConversionTracking::getConversionData('signup', ['user_id' => $vars['userid']]);
    
    $scripts = '';
    $scripts .= '<script>' . "\n";
    $scripts .= 'window.whmcsConversionData = ' . json_encode($tracking) . ';' . "\n";
    $scripts .= 'window.whmcsConversionType = "signup";' . "\n";
    $scripts .= 'document.dispatchEvent(new CustomEvent("whmcsConversion", {detail: window.whmcsConversionData}));' . "\n";
    $scripts .= '</script>' . "\n";
    
    return $scripts;
});
```

### Step 4: JavaScript Conversion Helpers

```javascript
/**
 * WHMCS Conversion Tracking JavaScript
 * Handles all conversion tracking events
 */

class WHMCSTracking {
    constructor() {
        this.listeners = [];
        this.setupEventListeners();
    }
    
    setupEventListeners() {
        // Listen for WHMCS conversion events
        document.addEventListener('whmcsConversion', (e) => {
            this.handleConversion(e.detail);
        });
        
        // Track page load conversions
        this.checkUrlParameters();
    }
    
    checkUrlParameters() {
        const urlParams = new URLSearchParams(window.location.search);
        
        // Google Ads conversion tracking
        if (urlParams.has('gclid')) {
            this.trackGoogleAdsConversion(urlParams.get('gclid'));
        }
        
        // Facebook pixel tracking
        if (urlParams.has('fbclid')) {
            this.trackFacebookClick(urlParams.get('fbclid'));
        }
        
        // UTM parameters for source attribution
        this.trackUTMSource(urlParams);
    }
    
    handleConversion(data) {
        const type = window.whmcsConversionType;
        
        switch (type) {
            case 'order':
                this.trackOrderConversion(data);
                break;
            case 'signup':
                this.trackSignupConversion(data);
                break;
            case 'ticket':
                this.trackTicketConversion(data);
                break;
        }
    }
    
    trackOrderConversion(data) {
        // Google Analytics E-commerce
        if (typeof gtag !== 'undefined') {
            gtag('event', 'purchase', {
                transaction_id: data.transaction_id,
                affiliation: data.affiliation,
                value: data.value,
                tax: data.tax,
                shipping: data.shipping,
                currency: data.currency,
                items: data.items
            });
        }
        
        // Google Ads conversion
        this.fireGoogleAdsConversion();
        
        // Facebook Purchase event
        if (typeof fbq !== 'undefined') {
            fbq('track', 'Purchase', {
                value: data.value,
                currency: data.currency,
                content_ids: data.items.map(i => i.sku),
                contents: data.items,
                content_type: 'product'
            });
        }
        
        // Custom event for other platforms
        this.dispatchCustomConversion('order_complete', data);
    }
    
    trackSignupConversion(data) {
        // Google Analytics event
        if (typeof gtag !== 'undefined') {
            gtag('event', 'sign_up', {
                method: 'Email'
            });
        }
        
        // Facebook CompleteRegistration
        if (typeof fbq !== 'undefined') {
            fbq('track', 'CompleteRegistration', {
                content_name: 'Client Registration',
                status: true
            });
        }
        
        this.dispatchCustomConversion('signup_complete', data);
    }
    
    trackTicketConversion(data) {
        if (typeof gtag !== 'undefined') {
            gtag('event', 'generate_lead', {
                method: 'Support Ticket'
            });
        }
        
        if (typeof fbq !== 'undefined') {
            fbq('track', 'Lead', {
                content_category: 'Support'
            });
        }
    }
    
    trackGoogleAdsConversion(gclid) {
        // Store GCLID for conversion tracking
        localStorage.setItem('whmcs_gclid', gclid);
        localStorage.setItem('whmcs_gclid_time', Date.now());
    }
    
    fireGoogleAdsConversion() {
        // Get Google Ads conversion ID and label from settings
        const conversionId = window.googleAdsConversionId;
        const conversionLabel = window.googleAdsConversionLabel;
        
        if (conversionId && conversionLabel) {
            // This would typically use the Google Ads conversion tracking script
            console.log('Google Ads conversion fired', {
                google_conversion_id: conversionId,
                google_conversion_label: conversionLabel,
                value: window.whmcsConversionValue,
                currency: window.whmcsConversionCurrency
            });
        }
    }
    
    trackFacebookClick(fbclid) {
        localStorage.setItem('whmcs_fbclid', fbclid);
        localStorage.setItem('whmcs_fbclid_time', Date.now());
    }
    
    trackUTMSource(params) {
        const utmData = {
            source: params.get('utm_source'),
            medium: params.get('utm_medium'),
            campaign: params.get('utm_campaign'),
            term: params.get('utm_term'),
            content: params.get('utm_content'),
            timestamp: Date.now()
        };
        
        // Store for later use on conversion
        if (utmData.source) {
            localStorage.setItem('whmcs_utm', JSON.stringify(utmData));
        }
    }
    
    dispatchCustomConversion(eventName, data) {
        // Custom event for third-party integrations
        document.dispatchEvent(new CustomEvent('whmcsCustomConversion', {
            detail: { event: eventName, data: data }
        }));
    }
}

// Initialize tracking
const whmcsTracking = new WHMCSTracking();

// Expose globally
window.whmcsTracking = whmcsTracking;
```

### Step 5: Server-Side Conversion Tracking

```php
<?php
/**
 * Server-side conversion tracking for sensitive data
 */
class ServerSideConversion {
    
    public static function trackOrder($orderId) {
        $config = self::getTrackingConfig();
        $data = ConversionTracking::getConversionData('order', ['order_id' => $orderId]);
        
        $events = [];
        
        // Google Ads server-side conversion
        if (!empty($config['google_ads_conversion_token'])) {
            $events[] = self::sendGoogleAdsConversion($data, $config);
        }
        
        // Facebook server-side events
        if (!empty($config['facebook_access_token'])) {
            $events[] = self::sendFacebookConversion($data, $config);
        }
        
        // Track conversion to database
        self::logConversion('order', $data);
        
        return $events;
    }
    
    private static function getTrackingConfig() {
        return [
            'google_ads_conversion_id' => \WHMCS\Config\Setting::getValue('GoogleAdsConversionId'),
            'google_ads_conversion_token' => \WHMCS\Config\Setting::getValue('GoogleAdsConversionToken'),
            'facebook_pixel_id' => \WHMCS\Config\Setting::getValue('FacebookPixelId'),
            'facebook_access_token' => \WHMCS\Config\Setting::getValue('FacebookConversionApiToken')
        ];
    }
    
    private static function sendGoogleAdsConversion($data, $config) {
        $url = 'https://www.google.com/pagead/conversion/' . $config['google_ads_conversion_id'] . '/';
        
        $postData = [
            'value' => $data['value'],
            'currency' => $data['currency'] ?? 'USD',
            'transaction_id' => $data['transaction_id'],
            '_fmt' => 'j'
        ];
        
        // Send via cURL
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($postData),
            CURLOPT_RETURNTRANSFER => true
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return ['platform' => 'google_ads', 'success' => !empty($response)];
    }
    
    private static function sendFacebookConversion($data, $config) {
        $pixelId = $config['facebook_pixel_id'];
        $accessToken = $config['facebook_access_token'];
        
        $eventData = [
            'event_name' => 'Purchase',
            'event_time' => time(),
            'action_source' => 'website',
            'event_source_url' => $_SERVER['HTTP_REFERER'] ?? '',
            'user_data' => [
                'client_ip_address' => $_SERVER['REMOTE_ADDR'],
                'client_user_agent' => $_SERVER['HTTP_USER_AGENT']
            ],
            'custom_data' => [
                'value' => $data['value'],
                'currency' => $data['currency'] ?? 'USD',
                'content_ids' => array_column($data['items'], 'sku'),
                'content_type' => 'product'
            ]
        ];
        
        $url = 'https://graph.facebook.com/v18.0/' . $pixelId . '/events';
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode(['data' => [$eventData]]),
            CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $accessToken],
            CURLOPT_RETURNTRANSFER => true
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return ['platform' => 'facebook', 'success' => json_decode($response, true)];
    }
    
    private static function logConversion($type, $data) {
        \WHMCS\Database\Capsule::table('mod_conversion_tracking')->insert([
            'type' => $type,
            'transaction_id' => $data['transaction_id'] ?? null,
            'data' => json_encode($data),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? null,
            'utm_source' => $_COOKIE['utm_source'] ?? null,
            'utm_medium' => $_COOKIE['utm_medium'] ?? null,
            'utm_campaign' => $_COOKIE['utm_campaign'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### Step 6: Hook for Server-Side Tracking

```php
<?php
/**
 * Trigger server-side tracking
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    // Queue server-side conversion for processing
    // This could be handled via queue/cron for reliability
    logActivity('Server-side conversion queued for order: ' . $vars['orderId']);
    
    // For immediate processing:
    // ServerSideConversion::trackOrder($vars['orderId']);
    
    return '';
});
```

### Step 7: Conversion Funnel Tracking

```javascript
/**
 * Track conversion funnel progression
 */
const funnelStages = {
    'product_view': 1,
    'add_to_cart': 2,
    'begin_checkout': 3,
    'payment_info': 4,
    'purchase_complete': 5
};

function trackFunnelStage(stage, data = {}) {
    const stageNumber = funnelStages[stage] || 0;
    
    if (typeof gtag !== 'undefined') {
        gtag('event', 'funnel_progression', {
            funnel_stage: stage,
            funnel_step: stageNumber,
            ...data
        });
    }
    
    // Store highest funnel stage in session
    const currentStage = sessionStorage.getItem('whmcs_funnel_stage') || 0;
    if (stageNumber > currentStage) {
        sessionStorage.setItem('whmcs_funnel_stage', stageNumber);
        sessionStorage.setItem('whmcs_funnel_' + stage, Date.now());
    }
}

// Track funnel events
document.addEventListener('DOMContentLoaded', function() {
    // Product view
    if (document.querySelector('[data-product-id]')) {
        const productId = document.querySelector('[data-product-id]').dataset.productId;
        trackFunnelStage('product_view', { product_id: productId });
    }
    
    // Add to cart buttons
    document.querySelectorAll('[data-action="add-to-cart"]').forEach(btn => {
        btn.addEventListener('click', function() {
            trackFunnelStage('add_to_cart', {
                product_id: this.dataset.productId
            });
        });
    });
    
    // Checkout initiation
    if (window.location.pathname.includes('checkout')) {
        trackFunnelStage('begin_checkout');
    }
    
    // Payment info entered
    if (document.querySelector('[data-track-payment]')) {
        document.querySelector('[data-track-payment]').addEventListener('click', function() {
            trackFunnelStage('payment_info');
        });
    }
});
```

### Step 8: Conversion Value Tracking

```php
<?php
/**
 * Calculate and track conversion values
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $valueScripts = '';
    
    // Lifetime Value tracking
    if ($vars['loggedin'] && isset($vars['client'])) {
        $client = $vars['client'];
        $userId = $client['id'];
        
        // Calculate client's lifetime value
        $lifetimeValue = self::calculateLifetimeValue($userId);
        
        $valueScripts .= '<script>' . "\n";
        $valueScripts .= 'window.whmcsClientLTV = ' . $lifetimeValue . ';' . "\n";
        $valueScripts .= 'window.whmcsClientId = "' . $userId . '";' . "\n";
        $valueScripts .= '</script>' . "\n";
    }
    
    return ['value_tracking' => $valueScripts];
});

private static function calculateLifetimeValue($userId) {
    // Get total spent
    $totalSpent = Capsule::table('tblinvoices')
        ->where('userid', $userId)
        ->where('status', 'Paid')
        ->sum('total');
    
    // Get account age in months
    $client = Capsule::table('tblclients')
        ->where('id', $userId)
        ->first();
    
    $accountAge = $client ? floor((time() - strtotime($client->datecreated)) / (30 * 24 * 60 * 60)) : 1;
    
    // Calculate monthly value
    $monthlyValue = $accountAge > 0 ? $totalSpent / $accountAge : $totalSpent;
    
    return json_encode([
        'total_spent' => $totalSpent,
        'account_age_months' => $accountAge,
        'monthly_value' => $monthlyValue,
        'projected_annual_value' => $monthlyValue * 12
    ]);
}
```

### Step 9: Conversion Reporting

```php
<?php
/**
 * Generate conversion reports
 */
function generateConversionReport($startDate, $endDate) {
    $conversions = Capsule::table('mod_conversion_tracking')
        ->whereBetween('created_at', [$startDate, $endDate])
        ->get();
    
    $report = [
        'total_conversions' => $conversions->count(),
        'by_type' => [],
        'by_utm_source' => [],
        'total_value' => 0
    ];
    
    foreach ($conversions as $conversion) {
        $data = json_decode($conversion->data, true);
        
        // Group by type
        $report['by_type'][$conversion->type] = ($report['by_type'][$conversion->type] ?? 0) + 1;
        
        // Group by UTM source
        if ($conversion->utm_source) {
            $report['by_utm_source'][$conversion->utm_source] = ($report['by_utm_source'][$conversion->utm_source] ?? 0) + 1;
        }
        
        // Sum total value
        $report['total_value'] += $data['value'] ?? 0;
    }
    
    return $report;
}
```

## Best Practices
- Track all conversion points in funnel
- Use server-side tracking for accuracy
- Implement proper attribution
- Test tracking with sandbox accounts
- Monitor for data discrepancies
- Use consistent event naming
- Track across all devices
- Consider view-through conversions
- Implement consent management
- Review attribution models regularly
