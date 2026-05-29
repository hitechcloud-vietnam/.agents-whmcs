# WHMCS Attribution Modeling Workflow

## Purpose
Implement multi-touch attribution modeling for WHMCS to understand marketing effectiveness.

## Prerequisites
- WHMCS installation
- Conversion tracking implemented
- Analytics platform
- Database access

## Step-by-Step Process

### Step 1: Create Attribution Tracking System

**Create hooks/attribution_tracking.php:**
```php
<?php
/**
 * WHMCS Attribution Modeling System
 * Tracks customer journey across touchpoints
 */

use WHMCS\Database\Capsule;

class AttributionTracker {
    
    const SOURCE_COOKIE = 'whmcs_attribution';
    const SOURCE_SESSION = 'whmcs_attribution_session';
    const REFERRAL_TABLE = 'mod_attribution_tracking';
    
    /**
     * Track attribution data on first touch
     */
    public static function trackFirstTouch($source = null, $medium = null, $campaign = null) {
        $data = [
            'first_source' => $source ?? $_GET['utm_source'] ?? $_SERVER['HTTP_REFERER'] ?? 'direct',
            'first_medium' => $medium ?? $_GET['utm_medium'] ?? 'none',
            'first_campaign' => $campaign ?? $_GET['utm_campaign'] ?? '',
            'first_term' => $_GET['utm_term'] ?? '',
            'first_content' => $_GET['utm_content'] ?? '',
            'first_touch_date' => date('Y-m-d H:i:s'),
            'gclid' => $_GET['gclid'] ?? '',
            'fbclid' => $_GET['fbclid'] ?? '',
            'msclkid' => $_GET['msclkid'] ?? '',
            'ip_address' => self::anonymizeIP($_SERVER['REMOTE_ADDR'] ?? ''),
            'user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500)
        ];
        
        $encoded = base64_encode(json_encode($data));
        
        // Store in cookie (30 days)
        setcookie(self::SOURCE_COOKIE, $encoded, time() + (30 * 24 * 60 * 60), '/', '', true, true);
        
        // Store in session
        $_SESSION[self::SOURCE_SESSION] = $data;
        
        return $data;
    }
    
    /**
     * Track subsequent touches
     */
    public static function trackTouch($type, $data = []) {
        $session = $_SESSION[self::SOURCE_SESSION] ?? [];
        
        if (empty($session)) {
            $session = self::trackFirstTouch();
        }
        
        // Update last touch info
        $session['last_touch_type'] = $type;
        $session['last_touch_date'] = date('Y-m-d H:i:s');
        $session['last_touch_url'] = $_SERVER['REQUEST_URI'] ?? '';
        
        // Track UTM updates if present
        if (isset($_GET['utm_source'])) {
            $session['last_source'] = $_GET['utm_source'];
            $session['last_medium'] = $_GET['utm_medium'] ?? '';
            $session['last_campaign'] = $_GET['utm_campaign'] ?? '';
            $session['last_touch_date'] = date('Y-m-d H:i:s');
        }
        
        // Update GCLID if present
        if (isset($_GET['gclid']) && empty($session['gclid'])) {
            $session['gclid'] = $_GET['gclid'];
        }
        
        $_SESSION[self::SOURCE_SESSION] = $session;
        
        return $session;
    }
    
    /**
     * Record conversion attribution
     */
    public static function recordConversion($userId, $orderId, $orderTotal) {
        $session = $_SESSION[self::SOURCE_SESSION] ?? [];
        
        if (empty($session)) {
            $session['first_source'] = 'unknown';
        }
        
        // Calculate days between first touch and conversion
        $firstTouchDate = strtotime($session['first_touch_date'] ?? date('Y-m-d H:i:s'));
        $conversionDate = time();
        $daysToConvert = floor(($conversionDate - $firstTouchDate) / (24 * 60 * 60));
        
        // Prepare attribution data
        $attributionData = [
            'user_id' => $userId,
            'order_id' => $orderId,
            'order_total' => $orderTotal,
            'first_source' => $session['first_source'] ?? 'unknown',
            'first_medium' => $session['first_medium'] ?? 'unknown',
            'first_campaign' => $session['first_campaign'] ?? '',
            'last_source' => $session['last_source'] ?? $session['first_source'] ?? 'unknown',
            'last_medium' => $session['last_medium'] ?? $session['first_medium'] ?? 'unknown',
            'last_campaign' => $session['last_campaign'] ?? '',
            'gclid' => $session['gclid'] ?? '',
            'fbclid' => $session['fbclid'] ?? '',
            'msclkid' => $session['msclkid'] ?? '',
            'first_touch_date' => $session['first_touch_date'] ?? null,
            'last_touch_date' => $session['last_touch_date'] ?? null,
            'days_to_convert' => $daysToConvert,
            'conversion_date' => date('Y-m-d H:i:s'),
            'ip_hash' => hash('sha256', $_SERVER['REMOTE_ADDR'] ?? ''),
            'user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500)
        ];
        
        // Save to database
        Capsule::table(self::REFERRAL_TABLE)->insert($attributionData);
        
        // Clear session attribution after conversion
        unset($_SESSION[self::SOURCE_SESSION]);
        
        return $attributionData;
    }
    
    /**
     * Get attribution data for current session
     */
    public static function getCurrentAttribution() {
        return $_SESSION[self::SOURCE_SESSION] ?? [];
    }
    
    /**
     * Anonymize IP address
     */
    private static function anonymizeIP($ip) {
        if (empty($ip)) return '';
        
        // Remove last octet for IPv4
        if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4)) {
            return preg_replace('/\.\d+$/', '.0', $ip);
        }
        
        return hash('sha256', $ip);
    }
    
    /**
     * Create tracking table if not exists
     */
    public static function createTable() {
        if (!Capsule::schema()->hasTable(self::REFERRAL_TABLE)) {
            Capsule::schema()->create(self::REFERRAL_TABLE, function($table) {
                $table->increments('id');
                $table->integer('user_id')->unsigned();
                $table->integer('order_id')->unsigned();
                $table->decimal('order_total', 10, 2);
                $table->string('first_source', 255);
                $table->string('first_medium', 255);
                $table->string('first_campaign', 255)->nullable();
                $table->string('last_source', 255);
                $table->string('last_medium', 255);
                $table->string('last_campaign', 255)->nullable();
                $table->string('gclid', 255)->nullable();
                $table->string('fbclid', 255)->nullable();
                $table->string('msclkid', 255)->nullable();
                $table->timestamp('first_touch_date')->nullable();
                $table->timestamp('last_touch_date')->nullable();
                $table->integer('days_to_convert')->default(0);
                $table->timestamp('conversion_date');
                $table->string('ip_hash', 64);
                $table->text('user_agent')->nullable();
                $table->timestamps();
                
                $table->index(['first_source']);
                $table->index(['last_source']);
                $table->index(['conversion_date']);
            });
        }
    }
}
```

### Step 2: Initialize Attribution Tracking

```php
<?php
/**
 * Initialize attribution tracking on page load
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    // Track page view attribution
    AttributionTracker::trackTouch('page_view', [
        'url' => $_SERVER['REQUEST_URI'] ?? '/',
        'page' => $vars['filename'] ?? 'unknown'
    ]);
    
    return [];
});
```

### Step 3: Record Conversion Attribution

```php
<?php
/**
 * Record attribution on order completion
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $userId = $vars['userid'] ?? 0;
    $orderId = $vars['orderId'] ?? 0;
    $amount = $vars['amount'] ?? 0;
    
    if ($userId && $orderId) {
        AttributionTracker::recordConversion($userId, $orderId, $amount);
    }
    
    return '';
});
```

### Step 4: Attribution Models

```php
<?php
/**
 * Calculate attribution for different models
 */

class AttributionModels {
    
    /**
     * First-touch attribution
     * 100% credit to first interaction
     */
    public static function firstTouch($startDate, $endDate) {
        $conversions = Capsule::table(AttributionTracker::REFERRAL_TABLE)
            ->whereBetween('conversion_date', [$startDate, $endDate])
            ->get();
        
        $attribution = [];
        
        foreach ($conversions as $conv) {
            $source = $conv->first_source;
            $attribution[$source] = ($attribution[$source] ?? 0) + $conv->order_total;
        }
        
        return $attribution;
    }
    
    /**
     * Last-touch attribution
     * 100% credit to last interaction
     */
    public static function lastTouch($startDate, $endDate) {
        $conversions = Capsule::table(AttributionTracker::REFERRAL_TABLE)
            ->whereBetween('conversion_date', [$startDate, $endDate])
            ->get();
        
        $attribution = [];
        
        foreach ($conversions as $conv) {
            $source = $conv->last_source;
            $attribution[$source] = ($attribution[$source] ?? 0) + $conv->order_total;
        }
        
        return $attribution;
    }
    
    /**
     * Linear attribution
     * Equal credit across all touchpoints
     */
    public static function linear($startDate, $endDate) {
        // This would need a touchpoints table for full implementation
        // Simplified version uses first and last
        $conversions = Capsule::table(AttributionTracker::REFERRAL_TABLE)
            ->whereBetween('conversion_date', [$startDate, $endDate])
            ->get();
        
        $attribution = [];
        
        foreach ($conversions as $conv) {
            $value = $conv->order_total / 2;
            $attribution[$conv->first_source] = ($attribution[$conv->first_source] ?? 0) + $value;
            $attribution[$conv->last_source] = ($attribution[$conv->last_source] ?? 0) + $value;
        }
        
        return $attribution;
    }
    
    /**
     * Time-decay attribution
     * More credit to recent touchpoints
     */
    public static function timeDecay($startDate, $endDate, $decayHalfLifeDays = 7) {
        $conversions = Capsule::table(AttributionTracker::REFERRAL_TABLE)
            ->whereBetween('conversion_date', [$startDate, $endDate])
            ->get();
        
        $attribution = [];
        
        foreach ($conversions as $conv) {
            $daysBetween = max(1, $conv->days_to_convert);
            $decayFactor = pow(0.5, $daysBetween / $decayHalfLifeDays);
            
            // First touch gets less credit over time
            $firstValue = $conv->order_total * $decayFactor * 0.3;
            $attribution[$conv->first_source] = ($attribution[$conv->first_source] ?? 0) + $firstValue;
            
            // Last touch gets more credit
            $lastValue = $conv->order_total * (1 - $decayFactor) * 0.7;
            $attribution[$conv->last_source] = ($attribution[$conv->last_source] ?? 0) + $lastValue;
        }
        
        return $attribution;
    }
    
    /**
     * Position-based attribution
     * 40% first touch, 40% last touch, 20% distributed
     */
    public static function positionBased($startDate, $endDate) {
        $conversions = Capsule::table(AttributionTracker::REFERRAL_TABLE)
            ->whereBetween('conversion_date', [$startDate, $endDate])
            ->get();
        
        $attribution = [];
        
        foreach ($conversions as $conv) {
            // First touch: 40%
            $firstValue = $conv->order_total * 0.4;
            $attribution[$conv->first_source] = ($attribution[$conv->first_source] ?? 0) + $firstValue;
            
            // Last touch: 40%
            $lastValue = $conv->order_total * 0.4;
            $attribution[$conv->last_source] = ($attribution[$conv->last_source] ?? 0) + $lastValue;
            
            // Middle touches: 20% (simplified, distributed to both)
            $middleValue = $conv->order_total * 0.2;
            $attribution[$conv->first_source] = ($attribution[$conv->first_source] ?? 0) + ($middleValue * 0.5);
            $attribution[$conv->last_source] = ($attribution[$conv->last_source] ?? 0) + ($middleValue * 0.5);
        }
        
        return $attribution;
    }
    
    /**
     * Generate attribution report
     */
    public static function generateReport($startDate, $endDate, $model = 'all') {
        $report = [
            'period' => ['start' => $startDate, 'end' => $endDate],
            'models' => []
        ];
        
        if ($model === 'all' || $model === 'first') {
            $report['models']['first_touch'] = self::firstTouch($startDate, $endDate);
        }
        
        if ($model === 'all' || $model === 'last') {
            $report['models']['last_touch'] = self::lastTouch($startDate, $endDate);
        }
        
        if ($model === 'all' || $model === 'linear') {
            $report['models']['linear'] = self::linear($startDate, $endDate);
        }
        
        if ($model === 'all' || $model === 'time_decay') {
            $report['models']['time_decay'] = self::timeDecay($startDate, $endDate);
        }
        
        if ($model === 'all' || $model === 'position') {
            $report['models']['position_based'] = self::positionBased($startDate, $endDate);
        }
        
        // Summary statistics
        $report['summary'] = [
            'total_conversions' => Capsule::table(AttributionTracker::REFERRAL_TABLE)
                ->whereBetween('conversion_date', [$startDate, $endDate])
                ->count(),
            'total_revenue' => Capsule::table(AttributionTracker::REFERRAL_TABLE)
                ->whereBetween('conversion_date', [$startDate, $endDate])
                ->sum('order_total'),
            'avg_days_to_convert' => Capsule::table(AttributionTracker::REFERRAL_TABLE)
                ->whereBetween('conversion_date', [$startDate, $endDate])
                ->avg('days_to_convert')
        ];
        
        return $report;
    }
}
```

### Step 5: JavaScript Attribution Helper

```javascript
/**
 * Client-side attribution tracking
 */
class AttributionHelper {
    
    constructor() {
        this.sessionId = this.getOrCreateSessionId();
        this.touchpoints = [];
        this.loadExistingTouchpoints();
    }
    
    getOrCreateSessionId() {
        let sessionId = sessionStorage.getItem('whmcs_session_id');
        if (!sessionId) {
            sessionId = this.generateUUID();
            sessionStorage.setItem('whmcs_session_id', sessionId);
        }
        return sessionId;
    }
    
    generateUUID() {
        return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
            const r = Math.random() * 16 | 0;
            const v = c === 'x' ? r : (r & 0x3 | 0x8);
            return v.toString(16);
        });
    }
    
    loadExistingTouchpoints() {
        const stored = sessionStorage.getItem('whmcs_touchpoints');
        if (stored) {
            try {
                this.touchpoints = JSON.parse(stored);
            } catch (e) {
                this.touchpoints = [];
            }
        }
    }
    
    saveTouchpoints() {
        sessionStorage.setItem('whmcs_touchpoints', JSON.stringify(this.touchpoints));
    }
    
    /**
     * Track a new touchpoint
     */
    trackTouchpoint(type, data = {}) {
        const touchpoint = {
            sessionId: this.sessionId,
            type: type,
            timestamp: new Date().toISOString(),
            url: window.location.href,
            referrer: document.referrer,
            utm_source: this.getUrlParam('utm_source'),
            utm_medium: this.getUrlParam('utm_medium'),
            utm_campaign: this.getUrlParam('utm_campaign'),
            utm_term: this.getUrlParam('utm_term'),
            utm_content: this.getUrlParam('utm_content'),
            gclid: this.getUrlParam('gclid'),
            fbclid: this.getUrlParam('fbclid'),
            msclkid: this.getUrlParam('msclkid'),
            page: this.getPageIdentifier(),
            ...data
        };
        
        this.touchpoints.push(touchpoint);
        this.saveTouchpoints();
        
        // Send to server
        this.sendTouchpoint(touchpoint);
    }
    
    getUrlParam(param) {
        const urlParams = new URLSearchParams(window.location.search);
        return urlParams.get(param) || '';
    }
    
    getPageIdentifier() {
        const path = window.location.pathname;
        const bodyId = document.body.id || '';
        return path + (bodyId ? '#' + bodyId : '');
    }
    
    sendTouchpoint(touchpoint) {
        // Send to server via beacon
        if (navigator.sendBeacon) {
            navigator.sendBeacon('/api/attribution/track', JSON.stringify(touchpoint));
        } else {
            fetch('/api/attribution/track', {
                method: 'POST',
                body: JSON.stringify(touchpoint),
                keepalive: true
            });
        }
    }
    
    /**
     * Track specific events
     */
    trackProductView(productId, productName) {
        this.trackTouchpoint('product_view', { productId, productName });
    }
    
    trackSearch(query) {
        this.trackTouchpoint('search', { query });
    }
    
    trackAddToCart(productId, productName, price) {
        this.trackTouchpoint('add_to_cart', { productId, productName, price });
    }
    
    trackBeginCheckout() {
        this.trackTouchpoint('begin_checkout');
    }
    
    trackConversion(orderId, total) {
        this.trackTouchpoint('conversion', { orderId, total });
        
        // Send all touchpoints for this session
        this.sendConversionData(orderId, total);
    }
    
    sendConversionData(orderId, total) {
        const data = {
            sessionId: this.sessionId,
            orderId: orderId,
            total: total,
            touchpoints: this.touchpoints,
            timestamp: new Date().toISOString()
        };
        
        if (navigator.sendBeacon) {
            navigator.sendBeacon('/api/attribution/conversion', JSON.stringify(data));
        } else {
            fetch('/api/attribution/conversion', {
                method: 'POST',
                body: JSON.stringify(data),
                keepalive: true
            });
        }
        
        // Clear touchpoints after conversion
        this.touchpoints = [];
        this.saveTouchpoints();
    }
}

// Initialize and expose globally
window.attributionHelper = new AttributionHelper();

// Track page view
document.addEventListener('DOMContentLoaded', function() {
    window.attributionHelper.trackTouchpoint('page_view');
    
    // Track product views
    document.querySelectorAll('[data-product-id]').forEach(function(el) {
        el.addEventListener('click', function() {
            window.attributionHelper.trackProductView(
                this.dataset.productId,
                this.dataset.productName || ''
            );
        });
    });
    
    // Track add to cart
    document.querySelectorAll('[data-action="add-to-cart"]').forEach(function(el) {
        el.addEventListener('click', function() {
            window.attributionHelper.trackAddToCart(
                this.dataset.productId,
                this.dataset.productName || '',
                parseFloat(this.dataset.price || 0)
            );
        });
    });
});
```

### Step 6: Generate Attribution Report

```php
<?php
/**
 * API endpoint for attribution reports
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    // Only for admin area reports
    if ($vars['filename'] !== 'admin') return '';
    
    return ['attribution_report' => ''];
});

// Usage in admin area
function getAttributionReportEndpoint($startDate, $endDate, $model = 'all') {
    return AttributionModels::generateReport($startDate, $endDate, $model);
}
```

## Best Practices
- Track all marketing touchpoints consistently
- Use server-side tracking for accuracy
- Implement multiple attribution models
- Consider view-through conversions
- Monitor for data discrepancies
- Use consistent UTM parameter naming
- Test attribution tracking regularly
- Consider assisted vs. direct conversions
- Review attribution windows
- Document attribution methodology
