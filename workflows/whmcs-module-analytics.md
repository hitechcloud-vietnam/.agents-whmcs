# WHMCS Module Analytics Workflow

## Description
Implement analytics collection for WHMCS modules.

## Steps

### Step 1: Create Analytics Service
```php
<?php
/**
 * Module Analytics Service
 */

class CLICodesAnalytics
{
    private $endpoint = 'https://analytics.example.com/collect';
    private $moduleId;
    private $moduleVersion;
    
    public function __construct($moduleId, $version)
    {
        $this->moduleId = $moduleId;
        $this->moduleVersion = $version;
    }
    
    /**
     * Track event
     */
    public function track($event, $data = [])
    {
        $payload = [
            'module_id' => $this->moduleId,
            'version' => $this->moduleVersion,
            'event' => $event,
            'data' => $data,
            'timestamp' => time(),
            'domain' => $_SERVER['HTTP_HOST'],
            'whmcs_version' => defined('WHMCS_VERSION') ? WHMCS_VERSION : 'unknown',
            'php_version' => PHP_VERSION,
        ];
        
        // Send asynchronously
        $this->sendAsync($payload);
    }
    
    /**
     * Track page view
     */
    public function trackPageView($page)
    {
        $this->track('page_view', ['page' => $page]);
    }
    
    /**
     * Track feature usage
     */
    public function trackFeatureUsage($feature, $action)
    {
        $this->track('feature_usage', [
            'feature' => $feature,
            'action' => $action,
        ]);
    }
    
    /**
     * Track error
     */
    public function trackError($error, $context = [])
    {
        $this->track('error', [
            'error' => $error,
            'context' => $context,
        ]);
    }
    
    /**
     * Send data async
     */
    private function sendAsync($data)
    {
        // Don't block the request
        ignore_user_abort(true);
        
        $ch = curl_init($this->endpoint);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 1,
        ]);
        
        curl_exec($ch);
        curl_close($ch);
    }
}
```

### Step 2: Integrate Analytics
```php
<?php
// In module file

$analytics = new CLICodesAnalytics('clicodes_example', '1.0.0');

function clicodes_example_output($vars)
{
    global $analytics;
    
    $analytics->trackPageView('dashboard');
    
    // Your module code...
}

add_hook('AfterModuleCreate', 1, function($vars) use ($analytics) {
    $analytics->trackFeatureUsage('module_create', 'success');
});
```

### Step 3: Create Analytics Dashboard
```php
<?php
// Admin analytics view

function clicodes_example_analytics($vars)
{
    return [
        'total_events' => getTotalEvents(),
        'popular_features' => getPopularFeatures(),
        'error_rate' => getErrorRate(),
        'active_installs' => getActiveInstalls(),
    ];
}
```

## Events to Track
| Event | Description | Data |
|-------|-------------|------|
| module_activated | Module activated | timestamp |
| module_configured | Settings saved | config_keys |
| feature_used | Feature accessed | feature_name |
| api_call | API call made | endpoint, status |
| error | Error occurred | error_type, message |

## Privacy Considerations
- Anonymize data where possible
- Allow opt-out
- Follow GDPR guidelines
- Don't collect sensitive data

## Tags
- analytics
- tracking
- metrics
- statistics