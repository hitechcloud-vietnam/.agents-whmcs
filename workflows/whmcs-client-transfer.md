# WHMCS Client Account Transfer Workflow

## Description
Transfer client accounts to different ownership.

## Steps

### Step 1: Transfer Request
```php
<?php
function transferClientAccount($clientId, $newOwnerEmail)
{
    // Verify new owner exists or create
    $newOwner = Capsule::table('tblclients')
        ->where('email', $newOwnerEmail)
        ->first();
    
    if (!$newOwner) {
        // Create new owner account
        $result = localAPI('AddClient', [
            'firstname' => 'New',
            'lastname' => 'Owner',
            'email' => $newOwnerEmail,
        ]);
        $newOwnerId = $result['clientid'];
    } else {
        $newOwnerId = $newOwner->id;
    }
    
    // Create transfer request
    Capsule::table('mod_transfer_log')->insert([
        'from_client_id' => $clientId,
        'to_client_id' => $newOwnerId,
        'reason' => $_POST['transfer_reason'] ?? '',
        'requested_by' => $_SESSION['adminid'],
        'requested_at' => date('Y-m-d H:i:s'),
        'status' => 'completed',
    ]);
    
    // Transfer ownership
    Capsule::table('tblhosting')
        ->where('userid', $clientId)
        ->update(['userid' => $newOwnerId]);
    
    Capsule::table('tbldomains')
        ->where('userid', $clientId)
        ->update(['userid' => $newOwnerId]);
    
    Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->update(['userid' => $newOwnerId]);
    
    Capsule::table('tblorders')
        ->where('userid', $clientId)
        ->update(['userid' => $newOwnerId]);
    
    // Send notifications
    sendEmail('AccountTransferred', $newOwnerId);
    sendEmail('AccountTransferredFrom', $clientId);
    
    logActivity("Client account transferred from $clientId to $newOwnerId");
}
```

## Tags
- transfer
- ownership-change
- account-move