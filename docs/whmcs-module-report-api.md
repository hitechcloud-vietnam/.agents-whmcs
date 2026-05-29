# WHMCS Report Module API

Complete reference for custom report module development in WHMCS.

## Module Structure

```php
<?php
/**
 * WHMCS Report Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function yourreport_MetaData()
{
    return [
        'name' => 'Your Custom Report',
        'description' => 'Generate custom reports',
        'version' => '1.0',
    ];
}

function yourreport_ConfigArray()
{
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'Your Report'],
    ];
}

function yourreport_report(array $params)
{
    // Build report data
    $data = buildReportData($params);
    
    return [
        'header' => ['Column1', 'Column2', 'Column3'],
        'data' => $data,
        'download' => true,
    ];
}

function buildReportData(array $params): array
{
    // Query and return data
    return Capsule::table('tblorders')
        ->select('id', 'ordernum', 'total')
        ->whereBetween('date', [$params['date_from'], $params['date_to']])
        ->get()
        ->toArray();
}
```

## Related Documentation

- [whmcs-functions-utility.md](../functions/whmcs-functions-utility.md)