# WHMCS Quote Builder Configuration Workflow

## Overview
Comprehensive workflow for configuring quotes and estimate generation in WHMCS.

## Prerequisites
- WHMCS v8.0+
- WHMCS Quotes enabled

## Step-by-Step Guide

### Step 1: Create Quote
```php
<?php
function create_quote(array $quoteData): int
{
    return \WHMCS\Database\Capsule::table('tblquotes')->insertGetId([
        'userid' => $quoteData['client_id'],
        'subject' => $quoteData['subject'],
        'validuntil' => $quoteData['valid_until'],
        'stage' => 'Draft',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}
```

### Step 2: Add Quote Line Items
```php
<?php
function add_quote_line(int $quoteId, array $item): int
{
    return \WHMCS\Database\Capsule::table('tblquoteitems')->insertGetId([
        'quoteid' => $quoteId,
        'description' => $item['description'],
        'quantity' => $item['quantity'],
        'unit_price' => $item['unit_price'],
        'discount' => $item['discount'] ?? 0,
    ]);
}
```

### Step 3: Quote Workflow Hook
```php
<?php
add_hook('QuoteAccept', 1, function($vars) {
    $quoteId = $vars['quoteid'];
    
    // Convert quote to order
    convert_quote_to_order($quoteId);
});
```

## Checklist
- Quote templates configured
- Default terms set
- Email notifications enabled
- Conversion workflow tested
