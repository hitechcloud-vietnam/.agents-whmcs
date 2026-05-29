# WHMCS Client Tags Automation Workflow

## Description
Automatically tag clients based on behavior and events.

## Steps

### Step 1: Create Tag System
```php
<?php
add_hook('OrderPaid', 1, function($vars) {
    $clientId = $vars['userId'];
    
    // Add tag based on order value
    $order = Capsule::table('tblorders')->find($vars['orderId']);
    if ($order->amount > 500) {
        addClientTag($clientId, 'high-value');
    }
    
    addClientTag($clientId, 'customer');
});

add_hook('TicketOpen', 1, function($vars) {
    addClientTag($vars['userid'], 'support-active');
});

function addClientTag($clientId, $tag)
{
    // Check if tag exists
    $existingTag = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->whereJsonContains('tags', $tag)
        ->first();
    
    if (!$existingTag) {
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update([
                'tags' => Capsule::raw("JSON_ARRAY_APPEND(tags, '$', '$tag')")
            ]);
    }
}
```

## Tags
- tagging
- segmentation
- automation
- classification