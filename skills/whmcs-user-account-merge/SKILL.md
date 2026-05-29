# WHMCS User Account Merge

## Overview
Guide for implementing account merging functionality in WHMCS. Covers merge requests, validation, and data consolidation.

## Account Merge Implementation

### Merge Request

```php
<?php
// /includes/hooks/account_merge.php

add_hook("AccountMergeRequest", 1, function(array $params) {
    $primaryClientId = $params["primary_client_id"];
    $secondaryClientId = $params["secondary_client_id"];
    
    // Validate both accounts exist
    $primary = Capsule::table("tblclients")->where("id", $primaryClientId)->first();
    $secondary = Capsule::table("tblclients")->where("id", $secondaryClientId)->first();
    
    if (!$primary || !$secondary) {
        return ["error" => "One or both accounts not found."];
    }
    
    // Check for conflicts
    $conflicts = checkMergeConflicts($primaryClientId, $secondaryClientId);
    if (!empty($conflicts)) {
        return ["error" => "Merge conflicts detected: " . implode(", ", $conflicts)];
    }
    
    // Generate merge token
    $token = bin2hex(random_bytes(32));
    
    Capsule::table("mod_account_merge_requests")->insert([
        "primary_client_id" => $primaryClientId,
        "secondary_client_id" => $secondaryClientId,
        "token" => hash("sha256", $token),
        "requested_at" => date("Y-m-d H:i:s"),
        "expires_at" => date("Y-m-d H:i:s", strtotime("+7 days")),
        "status" => "pending"
    ]);
    
    // Send notification to secondary account owner
    send_email("AccountMergeRequest", $secondaryClientId, [
        "primary_email" => $primary->email,
        "primary_name" => $primary->firstname . " " . $primary->lastname,
        "approve_link" => "merge-accounts.php?action=approve&token=" . $token,
        "reject_link" => "merge-accounts.php?action=reject&token=" . $token
    ]);
    
    return [
        "success" => true,
        "message" => "Merge request sent. Waiting for confirmation."
    ];
});

function checkMergeConflicts(int $primaryId, int $secondaryId): array
{
    $conflicts = [];
    
    // Check for active overlapping services
    $secondaryServices = Capsule::table("tblhosting")
        ->where("userid", $secondaryId)
        ->whereIn("domainstatus", ["Active", "Suspended"])
        ->get();
    
    foreach ($secondaryServices as $service) {
        $primaryService = Capsule::table("tblhosting")
            ->where("userid", $primaryId)
            ->where("domain", $service->domain)
            ->first();
        
        if ($primaryService) {
            $conflicts[] = "Duplicate domain: " . $service->domain;
        }
    }
    
    // Check for conflicting email addresses
    $primaryEmails = getContactEmails($primaryId);
    $secondaryEmails = getContactEmails($secondaryId);
    
    $overlap = array_intersect($primaryEmails, $secondaryEmails);
    if (!empty($overlap)) {
        $conflicts[] = "Overlapping contact emails: " . implode(", ", $overlap);
    }
    
    return $conflicts;
}
```

### Process Merge

```php
add_hook("ProcessAccountMerge", 1, function(array $params) {
    $primaryClientId = $params["primary_client_id"];
    $secondaryClientId = $params["secondary_client_id"];
    
    // Begin transaction
    Capsule::connection()->transaction(function() use ($primaryClientId, $secondaryClientId) {
        // Merge services
        Capsule::table("tblhosting")
            ->where("userid", $secondaryClientId)
            ->update(["userid" => $primaryClientId]);
        
        // Merge domains
        Capsule::table("tbldomains")
            ->where("userid", $secondaryClientId)
            ->update(["userid" => $primaryClientId]);
        
        // Merge orders
        Capsule::table("tblorders")
            ->where("userid", $secondaryClientId)
            ->update(["userid" => $primaryClientId]);
        
        // Merge invoices
        Capsule::table("tblinvoices")
            ->where("userid", $secondaryClientId)
            ->update(["userid" => $primaryClientId]);
        
        // Merge addons
        Capsule::table("tblhostingaddons")
            ->where("hostingid", function($query) use ($secondaryClientId) {
                $query->select("id")
                      ->from("tblhosting")
                      ->where("userid", $secondaryClientId);
            })
            ->update(["hostingid" => function($query) use ($primaryClientId) {
                // Update logic handled separately
            });
        
        // Merge tickets
        Capsule::table("tbltickets")
            ->where("userid", $secondaryClientId)
            ->update(["userid" => $primaryClientId]);
        
        // Merge tickets (admin tickets by email)
        Capsule::table("tbltickets")
            ->where("userid", 0)
            ->whereIn("email", getContactEmails($secondaryClientId))
            ->update(["userid" => $primaryClientId]);
        
        // Merge transactions
        Capsule::table("tblaccounts")
            ->where("userid", $secondaryClientId)
            ->update(["userid" => $primaryClientId]);
        
        // Merge credits
        $secondaryCredit = getAccountCredit($secondaryClientId);
        if ($secondaryCredit > 0) {
            addCredit($primaryClientId, $secondaryCredit, "Merged from account #" . $secondaryClientId);
        }
        
        // Merge custom field values
        $customFields = Capsule::table("tblcustomfieldsvalues")
            ->where("relid", $secondaryClientId)
            ->get();
        
        foreach ($customFields as $field) {
            $existing = Capsule::table("tblcustomfieldsvalues")
                ->where("relid", $primaryClientId)
                ->where("fieldid", $field->fieldid)
                ->first();
            
            if ($existing) {
                // Skip if already exists
            } else {
                Capsule::table("tblcustomfieldsvalues")
                    ->where("id", $field->id)
                    ->update(["relid" => $primaryClientId]);
            }
        }
        
        // Archive secondary account
        Capsule::table("tblclients")
            ->where("id", $secondaryClientId)
            ->update([
                "email" => "merged_" . $secondaryClientId . "@archived.local",
                "status" => "Closed",
                "merged_into" => $primaryClientId,
                "merged_at" => date("Y-m-d H:i:s")
            ]);
        
        // Log merge
        Capsule::table("mod_account_merges")->insert([
            "primary_client_id" => $primaryClientId,
            "secondary_client_id" => $secondaryClientId,
            "merged_at" => date("Y-m-d H:i:s")
        ]);
    });
    
    return ["success" => true];
});
```

## Merge Template

```smarty
<!-- /templates/merge_accounts.tpl -->
<div class="merge-container">
    <div class="merge-card">
        <h2>Merge Accounts</h2>
        <p>Combine two accounts into one. The secondary account will be archived.</p>
        
        <form method="post" action="merge-accounts.php" class="merge-form">
            <input type="hidden" name="token" value="{$token}">
            
            <div class="merge-section">
                <h3>Primary Account (will be kept)</h3>
                <div class="form-group">
                    <label for="primary_email">Email Address</label>
                    <input type="email" name="primary_email" id="primary_email" 
                           required class="form-control"
                           placeholder="your@email.com">
                </div>
                <div class="form-group">
                    <label for="primary_password">Password</label>
                    <input type="password" name="primary_password" 
                           required class="form-control">
                </div>
            </div>
            
            <div class="merge-section">
                <h3>Secondary Account (will be archived)</h3>
                <div class="form-group">
                    <label for="secondary_email">Email Address</label>
                    <input type="email" name="secondary_email" 
                           id="secondary_email" required class="form-control"
                           placeholder="your@email.com">
                </div>
                <div class="form-group">
                    <label for="secondary_password">Password</label>
                    <input type="password" name="secondary_password" 
                           required class="form-control">
                </div>
            </div>
            
            <div class="info-box">
                <h4>What will happen:</h4>
                <ul>
                    <li>All services from both accounts will be combined</li>
                    <li>Invoice and payment history will be preserved</li>
                    <li>Support tickets will be transferred</li>
                    <li>Secondary account email will be deactivated</li>
                    <li>You will receive a confirmation email</li>
                </ul>
            </div>
            
            <button type="submit" class="btn btn-primary btn-block">
                Request Account Merge
            </button>
        </form>
    </div>
</div>
```

## Best Practices

1. **Validation**: Verify ownership of both accounts
2. **Conflict Detection**: Identify and report conflicts
3. **Data Integrity**: Use transactions for atomic merges
4. **Service Handling**: Properly transfer all services
5. **Credit Transfer**: Move account credits
6. **Notification**: Email both account owners
7. **Audit Trail**: Log all merge operations
8. **Rollback**: Keep secondary account data temporarily
9. **Email Handling**: Update contact email associations
