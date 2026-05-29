# WHMCS Widget Module API

Complete reference for dashboard widget module development in WHMCS.

## Module Structure

```php
<?php
/**
 * WHMCS Widget Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function yourwidget_MetaData()
{
    return [
        'DisplayName' => 'Your Widget',
        'Description' => 'A custom dashboard widget',
    ];
}

function yourwidget_ConfigArray()
{
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'Your Widget'],
        'RefreshInterval' => [
            'FriendlyName' => 'Refresh Interval (seconds)',
            'Type' => 'text',
            'Default' => '60',
        ],
    ];
}

function yourwidget_output(array $params)
{
    // Get widget data
    $data = getWidgetData();
    
    return [
        'templatefile' => 'widget_template',
        'vars' => [
            'title' => 'Your Widget',
            'data' => $data,
        ],
    ];
}
```

## Related Documentation

- [whmcs-advanced-caching.md](../advanced/whmcs-advanced-caching.md)