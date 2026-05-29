# WHMCS Widget Debug Workflow

## Overview
This workflow guides you through debugging WHMCS dashboard widget issues.

## Prerequisites
- Widget implementation
- Browser developer tools

## Step-by-Step Guide

### Step 1: Check Widget Registration
```php
// In your module
function yourmodule_output()
{
    // Register widget
    add_hook('AdminHomeWidgets', 1, function() {
        return new \WHMCS\Module\YourModule\Widget();
    });
}
```

### Step 2: Enable Widget Debugging
```php
// In your widget class
class StatisticsWidget extends \WHMCS\Module\AbstractWidget
{
    public function getTitle(): string
    {
        return 'Statistics';
    }

    public function getBody(): string
    {
        try {
            $data = $this->getData();
            return $this->renderTemplate($data);
        } catch (Exception $e) {
            logModuleCall(
                'yourmodule',
                'widget_error',
                get_class($this),
                $e->getMessage()
            );
            return '<div class="alert alert-danger">Error loading widget</div>';
        }
    }

    private function getData(): array
    {
        // Fetch and return data
        $this->log->debug("Fetching widget data");
        return [
            'total_clients' => \WHMCS\Database\Capsule::table('tblclients')->count(),
            'active_services' => \WHMCS\Database\Capsule::table('tblhosting')
                ->where('domainstatus', 'Active')
                ->count(),
        ];
    }
}
```

### Step 3: Check Widget in Browser
```javascript
// Open browser console
// Navigate to WHMCS Admin Home Dashboard
// Check Network tab for widget API calls
// Check Console for JavaScript errors
```

### Step 4: Common Widget Issues
```php
// Issue: Widget not appearing
// Fix: Check module activated, hook registered

// Issue: Widget empty
// Fix: Check data fetch logic, log errors

// Issue: Widget slow
// Fix: Cache data, optimize queries
```

## Widget Debug Checklist

### Investigation
- [ ] Widget registered
- [ ] Module activated
- [ ] Console errors checked
- [ ] Data fetched

### Resolution
- [ ] Hook fixed
- [ ] Data logic corrected
- [ ] Template fixed
- [ ] Performance optimized
