# WHMCS User Account Deletion

## Overview
Guide for implementing user account deletion in WHMCS. Covers GDPR compliance, data retention, and deletion workflows.

## Account Deletion Flow

### Request Deletion

```php
<?php
// /includes/hooks/account_deletion.php

add_hook("AccountDeletionRequest", 1, function(array $params) {
    $clientId = $params["client_id"];
    
    // Check for active services
    $activeServices = Capsule::table("tblhosting")
        ->where("userid", $clientId)
        ->whereIn("domainstatus", ["Active", "Suspended"])
        ->count();
    
    if ($activeServices > 0) {
        return [
            "error" => "Cannot delete account with active services. 
                       Please cancel all services first."
        ];
    }
    
    // Check pending invoices
    $pendingInvoices = Capsule::table("tblinvoices")
        ->where("userid", $clientId)
        ->whereIn("status", ["Unpaid", "Draft"])
        ->count();
    
    if ($pendingInvoices > 0) {
        return [
            "error" => "Please settle pending invoices before deletion."
        ];
    }
    
    // Generate deletion token
    $token = bin2hex(random_bytes(32));
    
    Capsule::table("mod_deletion_requests")->insert([
        "client_id" => $clientId,
        "token" => hash("sha256", $token),
        "requested_at" => date("Y-m-d H:i:s"),
        "scheduled_at" => date("Y-m-d H:i:s", strtotime("+30 days")),
        "status" => "pending"
    ]);
    
    // Send confirmation email
    send_email("AccountDeletionRequest", $clientId, [
        "deletion_date" => date("Y-m-d", strtotime("+30 days")),
        "cancel_link" => "cancel-deletion.php?token=" . $token
    ]);
    
    return [
        "success" => true,
        "message" => "Deletion scheduled. You will receive confirmation email."
    ];
});
```

### Process Deletion

```php
add_hook("ProcessAccountDeletion", 1, function(array $params) {
    $clientId = $params["client_id"];
    
    // Backup data
    $backupData = backupClientData($clientId);
    
    // Anonymize personal data
    Capsule::table("tblclients")
        ->where("id", $clientId)
        ->update([
            "firstname" => "DELETED",
            "lastname" => "USER",
            "companyname" => "",
            "email" => "deleted_" . $clientId . "@anonymized.local",
            "address1" => "",
            "address2" => "",
            "city" => "",
            "state" => "",
            "postcode" => "",
            "country" => "",
            "phonenumber" => "",
            "password" => password_hash(bin2hex(random_bytes(32)), PASSWORD_DEFAULT),
            "status" => "Closed",
            "deleted_at" => date("Y-m-d H:i:s")
        ]);
    
    // Remove custom fields
    Capsule::table("tblcustomfieldsvalues")
        ->where("relid", $clientId)
        ->delete();
    
    // Handle related records
    handleDeletedUserRecords($clientId);
    
    // Log deletion
    Capsule::table("mod_account_deletions")->insert([
        "client_id" => $clientId,
        "backup_id" => $backupData["id"],
        "deleted_at" => date("Y-m-d H:i:s"),
        "deleted_by" => $params["deleted_by"] ?? "user",
        "reason" => $params["reason"] ?? ""
    ]);
    
    return ["success" => true];
});

function handleDeletedUserRecords(int $clientId): void
{
    // Close open tickets
    Capsule::table("tbltickets")
        ->where("userid", $clientId)
        ->update(["status" => "Closed"]);
    
    // Cancel pending orders
    Capsule::table("tblorders")
        ->where("userid", $clientId)
        ->whereIn("status", ["Pending", "Active"])
        ->update(["status" => "Cancelled"]);
    
    // Archive transactions (for tax compliance)
    // Keep for required retention period
}
```

### Cancel Deletion

```php
add_hook("CancelAccountDeletion", 1, function(array $params) {
    $token = $params["token"];
    
    $request = Capsule::table("mod_deletion_requests")
        ->where("token", hash("sha256", $token))
        ->where("status", "pending")
        ->first();
    
    if (!$request) {
        return ["error" => "Invalid or expired cancellation request."];
    }
    
    Capsule::table("mod_deletion_requests")
        ->where("id", $request->id)
        ->update([
            "status" => "cancelled",
            "cancelled_at" => date("Y-m-d H:i:s")
        ]);
    
    // Send confirmation
    send_email("AccountDeletionCancelled", $request->client_id, []);
    
    return ["success" => true];
});
```

## Deletion Request Template

```smarty
<!-- /templates/account_deletion.tpl -->
<div class="deletion-container">
    <div class="deletion-card">
        <h2>Delete Your Account</h2>
        
        <div class="warning-box">
            <i class="fa fa-exclamation-triangle"></i>
            <h3>Warning: This action cannot be undone</h3>
            <p>The following will be permanently deleted:</p>
            <ul>
                <li>Personal information and profile data</li>
                <li>Account settings and preferences</li>
                <li>Custom fields and additional data</li>
                <li>Login history and activity logs</li>
            </ul>
        </div>
        
        <div class="info-box">
            <h3>What will be retained:</h3>
            <ul>
                <li>Invoice records (legal requirement)</li>
                <li>Transaction history (tax compliance)</li>
                <li>Service termination records</li>
            </ul>
        </div>
        
        {if $hasActiveServices}
            <div class="alert alert-warning">
                <p>You have active services that must be cancelled first:</p>
                <ul>
                    {foreach $activeServices as $service}
                        <li>{$service->domain} ({$service->productname})</li>
                    {/foreach}
                </ul>
                <a href="clientarea.php?action=cancel" class="btn btn-secondary">
                    Cancel Services
                </a>
            </div>
        {else}
            <form method="post" action="delete-account.php" class="deletion-form">
                <input type="hidden" name="token" value="{$token}">
                <input type="hidden" name="action" value="request">
                
                <div class="form-group">
                    <label for="password">Confirm with your password</label>
                    <input type="password" name="password" id="password" 
                           required class="form-control">
                </div>
                
                <div class="form-group">
                    <label for="reason">Reason for deletion (optional)</label>
                    <select name="reason" id="reason" class="form-control">
                        <option value="">Select a reason...</option>
                        <option value="not_needed">No longer needed</option>
                        <option value="switching">Switching to another provider</option>
                        <option value="privacy">Privacy concerns</option>
                        <option value="expensive">Too expensive</option>
                        <option value="other">Other</option>
                    </select>
                </div>
                
                <div class="form-group checkbox">
                    <label>
                        <input type="checkbox" name="confirm" value="1" required>
                        I understand this action is permanent and irreversible
                    </label>
                </div>
                
                <button type="submit" class="btn btn-danger btn-block">
                    <i class="fa fa-trash"></i>
                    Request Account Deletion
                </button>
            </form>
        {/if}
        
        <div class="deletion-footer">
            <p>Need help? <a href="support.php">Contact Support</a></p>
        </div>
    </div>
</div>
```

## Best Practices

1. **GDPR Compliance**: Honor right to deletion requests
2. **Grace Period**: Allow cancellation during waiting period
3. **Service Check**: Prevent deletion with active services
4. **Invoice Check**: Ensure no outstanding balances
5. **Data Backup**: Archive data before deletion
6. **Data Anonymization**: Anonymize rather than delete when possible
7. **Retention**: Keep financial records for legal compliance
8. **Confirmation**: Require password confirmation
9. **Notifications**: Email confirmations at each step
