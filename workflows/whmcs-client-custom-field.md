# WHMCS Client Custom Field Setup Workflow

## Description
Configure and use custom fields for client data collection.

## Steps

### Step 1: Create Custom Fields
```php
<?php
function createClientCustomFields()
{
    $fields = [
        [
            'name' => 'VAT Number',
            'type' => 'text',
            'required' => 1,
            'show_on_invoice' => 1,
        ],
        [
            'name' => 'Industry',
            'type' => 'dropdown',
            'options' => 'Technology,Healthcare,Finance,Retail,Other',
            'required' => 0,
        ],
        [
            'name' => 'Newsletter',
            'type' => 'tickbox',
            'required' => 0,
        ],
    ];
    
    foreach ($fields as $field) {
        Capsule::table('tblcustomfields')->insert([
            'type' => 'client',
            'fieldname' => $field['name'],
            'fieldtype' => $field['type'],
            'fieldoptions' => $field['options'] ?? '',
            'required' => $field['required'] ?? 0,
            'showinvoice' => $field['show_on_invoice'] ?? 0,
        ]);
    }
}
```

### Step 2: Access Custom Field Values
```php
<?php
function getClientCustomFields($clientId)
{
    return Capsule::table('tblcustomfieldsvalues')
        ->join('tblcustomfields', 'tblcustomfieldsvalues.fieldid', '=', 'tblcustomfields.id')
        ->where('tblcustomfieldsvalues.relid', $clientId)
        ->select('tblcustomfields.fieldname', 'tblcustomfieldsvalues.value')
        ->get();
}
```

## Tags
- custom-fields
- client-data
- data-collection
- forms