# WHMCS Client Deletion Workflow

## Description
Properly handle client account deletion in WHMCS (GDPR compliance).

## Prerequisites
- WHMCS 7.0+
- Legal basis for deletion (GDPR, etc.)

## Steps

### Step 1: Verify Deletion Eligibility
```php
<?php
function canDeleteClient($clientId)
{
    // Check for pending orders
    $pendingOrders = Capsule::table('tblorders')
        ->where('userid', $clientId)
        ->whereIn('status', ['Pending', 'Active'])
        ->count();
    
    if ($pendingOrders > 0) {
        return ['can_delete' => false, 'reason' => 'Pending orders exist'];
    }
    
    // Check for unpaid invoices
    $unpaidInvoices = Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->where('status', 'Unpaid')
        ->count();
    
    if ($unpaidInvoices > 0) {
        return ['can_delete' => false, 'reason' => 'Unpaid invoices'];
    }
    
    return ['can_delete' => true];
}
```

### Step 2: Anonymize Client Data
```php
<?php
/**
 * GDPR-compliant client deletion
 * Anonymizes personal data instead of hard delete
 */

function anonymizeClient($clientId)
{
    // Anonymize main client record
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update([
            'firstname' => 'Deleted',
            'lastname' => 'User',
            'email' => 'deleted_' . $clientId . '@anonymized.local',
            'companyname' => '',
            'address1' => '',
            'address2' => '',
            'city' => '',
            'state' => '',
            'postcode' => '',
            'country' => '',
            'phonenumber' => '',
            'phonenumbercc' => '',
            'password' => hash('sha256', bin2hex(random_bytes(32))),
            'authdata' => '',
            // Keep for reference
            'anonymized_at' => date('Y-m-d H:i:s'),
            'anonymized_reason' => 'GDPR deletion request',
        ]);
    
    // Anonymize activity logs (keep for auditing)
    Capsule::table('tblactivitylog')
        ->where('userid', $clientId)
        ->update(['userid' => 0]);
    
    // Log the deletion
    logActivity("Client data anonymized for GDPR compliance: ID $clientId");
    
    return true;
}
```

### Step 3: Handle Related Data
```php
<?php
function deleteOrAnonymizeRelatedData($clientId)
{
    // Delete service relationships
    Capsule::table('tblhosting')
        ->where('userid', $clientId)
        ->delete();
    
    // Delete domains
    Capsule::table('tbldomains')
        ->where('userid', $clientId)
        ->delete();
    
    // Keep invoices for tax compliance (anonymize)
    Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->update([
            'userid' => 0,
            'name' => 'Anonymized Customer',
        ]);
    
    // Delete tickets (optional - may need to keep)
    Capsule::table('tbltickets')
        ->where('userid', $clientId)
        ->delete();
    
    // Delete custom field values
    Capsule::table('tblcustomfieldsvalues')
        ->where('relid', $clientId)
        ->where('fieldid', function($q) {
            $q->from('tblcustomfields')
              ->where('type', 'client');
        })
        ->delete();
}
```

### Step 4: Admin Deletion Workflow
```php
<?php
// In admin area

function adminDeleteClient($clientId, $reason)
{
    $check = canDeleteClient($clientId);
    
    if (!$check['can_delete']) {
        return ['error' => $check['reason']];
    }
    
    // Get client email for records
    $client = Capsule::table('tblclients')->find($clientId);
    $deletionLog = "Deleted: $client->email, Reason: $reason, Admin: " . $_SESSION['adminid'];
    
    // Perform deletion
    anonymizeClient($clientId);
    deleteOrAnonymizeRelatedData($clientId);
    
    // Log for compliance
    Capsule::table('mod_deletion_log')->insert([
        'client_id' => $clientId,
        'original_email_hash' => hash('sha256', $client->email),
        'reason' => $reason,
        'deleted_by' => $_SESSION['adminid'],
        'deleted_at' => date('Y-m-d H:i:s'),
    ]);
    
    return ['success' => true];
}
```

## Tags
- deletion
- gdpr
- privacy
- compliance
- anonymization