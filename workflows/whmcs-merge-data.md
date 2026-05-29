# WHMCS Data Merge Workflow

## Purpose
Merge duplicate client accounts and data in WHMCS.

## Prerequisites
- WHMCS installation
- Admin access
- Backup completed

## Step-by-Step Process

### Step 1: Create Data Merge Handler

**Create hooks/data_merge.php:**
```php
<?php
/**
 * WHMCS Data Merge Handler
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class DataMerge {
    
    /**
     * Find potential duplicate clients
     */
    public function findDuplicates($criteria = 'email') {
        $duplicates = [];
        
        switch ($criteria) {
            case 'email':
                $duplicates = $this->findEmailDuplicates();
                break;
            case 'name':
                $duplicates = $this->findNameDuplicates();
                break;
            case 'phone':
                $duplicates = $this->findPhoneDuplicates();
                break;
        }
        
        return $duplicates;
    }
    
    /**
     * Find email duplicates
     */
    private function findEmailDuplicates() {
        $query = "SELECT email, COUNT(*) as count 
                  FROM tblclients 
                  GROUP BY email 
                  HAVING count > 1";
        
        $duplicates = Capsule::select($query);
        
        $results = [];
        foreach ($duplicates as $dup) {
            $clients = Capsule::table('tblclients')
                ->where('email', $dup->email)
                ->get();
            
            $results[] = [
                'email' => $dup->email,
                'count' => $dup->count,
                'clients' => $clients->toArray()
            ];
        }
        
        return $results;
    }
    
    /**
     * Find name duplicates
     */
    private function findNameDuplicates() {
        $query = "SELECT firstname, lastname, COUNT(*) as count 
                  FROM tblclients 
                  GROUP BY firstname, lastname 
                  HAVING count > 1";
        
        $duplicates = Capsule::select($query);
        
        $results = [];
        foreach ($duplicates as $dup) {
            $clients = Capsule::table('tblclients')
                ->where('firstname', $dup->firstname)
                ->where('lastname', $dup->lastname)
                ->get();
            
            $results[] = [
                'name' => $dup->firstname . ' ' . $dup->lastname,
                'count' => $dup->count,
                'clients' => $clients->toArray()
            ];
        }
        
        return $results;
    }
    
    /**
     * Find phone duplicates
     */
    private function findPhoneDuplicates() {
        $query = "SELECT phonenumber, COUNT(*) as count 
                  FROM tblclients 
                  WHERE phonenumber != '' 
                  GROUP BY phonenumber 
                  HAVING count > 1";
        
        $duplicates = Capsule::select($query);
        
        $results = [];
        foreach ($duplicates as $dup) {
            $clients = Capsule::table('tblclients')
                ->where('phonenumber', $dup->phonenumber)
                ->get();
            
            $results[] = [
                'phone' => $dup->phonenumber,
                'count' => $dup->count,
                'clients' => $clients->toArray()
            ];
        }
        
        return $results;
    }
    
    /**
     * Merge two client accounts
     */
    public function mergeClients($primaryId, $secondaryId, $options = []) {
        $defaults = [
            'keep_primary_services' => true,
            'keep_primary_invoices' => true,
            'archive_secondary' => true,
            'transfer_services' => true,
            'transfer_domains' => true,
            'transfer_tickets' => true
        ];
        $options = array_merge($defaults, $options);
        
        $primary = Capsule::table('tblclients')->where('id', $primaryId)->first();
        $secondary = Capsule::table('tblclients')->where('id', $secondaryId)->first();
        
        if (!$primary || !$secondary) {
            throw new Exception("One or both clients not found");
        }
        
        // Start transaction
        Capsule::beginTransaction();
        
        try {
            // Transfer services
            if ($options['transfer_services']) {
                $this->transferServices($secondaryId, $primaryId);
            }
            
            // Transfer domains
            if ($options['transfer_domains']) {
                $this->transferDomains($secondaryId, $primaryId);
            }
            
            // Transfer tickets
            if ($options['transfer_tickets']) {
                $this->transferTickets($secondaryId, $primaryId);
            }
            
            // Merge orders
            $this->mergeOrders($secondaryId, $primaryId);
            
            // Merge activity logs
            $this->mergeActivityLogs($secondaryId, $primaryId);
            
            // Archive secondary account
            if ($options['archive_secondary']) {
                $this->archiveClient($secondaryId);
            }
            
            // Log merge operation
            $this->logMergeOperation($primaryId, $secondaryId, $options);
            
            Capsule::commit();
            
            return [
                'success' => true,
                'primary_id' => $primaryId,
                'secondary_id' => $secondaryId
            ];
            
        } catch (Exception $e) {
            Capsule::rollBack();
            throw $e;
        }
    }
    
    /**
     * Transfer services to primary account
     */
    private function transferServices($fromId, $toId) {
        Capsule::table('tblhosting')
            ->where('userid', $fromId)
            ->update(['userid' => $toId]);
    }
    
    /**
     * Transfer domains to primary account
     */
    private function transferDomains($fromId, $toId) {
        Capsule::table('tbldomains')
            ->where('userid', $fromId)
            ->update(['userid' => $toId]);
    }
    
    /**
     * Transfer tickets to primary account
     */
    private function transferTickets($fromId, $toId) {
        Capsule::table('tbltickets')
            ->where('userid', $fromId)
            ->update(['userid' => $toId]);
        
        Capsule::table('tblticketreplies')
            ->where('userid', $fromId)
            ->update(['userid' => $toId]);
    }
    
    /**
     * Merge orders
     */
    private function mergeOrders($fromId, $toId) {
        // Update orders to point to new user
        Capsule::table('tblorders')
            ->where('userid', $fromId)
            ->update(['userid' => $toId]);
    }
    
    /**
     * Merge activity logs
     */
    private function mergeActivityLogs($fromId, $toId) {
        // Mark secondary logs with note about merge
        $note = ' [Merged from client ID: ' . $fromId . ']';
        
        Capsule::table('tblactivitylog')
            ->where('userid', $fromId)
            ->update(['userid' => $toId]);
    }
    
    /**
     * Archive client account
     */
    private function archiveClient($clientId) {
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update([
                'email' => 'archived_' . $clientId . '_' . time() . '@deleted.local',
                'status' => 'Closed',
                'notes' => Capsule::raw("CONCAT(COALESCE(notes, ''), ' [Archived: " . Carbon::now()->toDateTimeString() . "]')")
            ]);
    }
    
    /**
     * Log merge operation
     */
    private function logMergeOperation($primaryId, $secondaryId, $options) {
        Capsule::table('mod_merge_logs')->insert([
            'primary_id' => $primaryId,
            'secondary_id' => $secondaryId,
            'options' => json_encode($options),
            'performed_by' => $_SESSION['adminid'] ?? null,
            'created_at' => Carbon::now()->toDateTimeString()
        ]);
    }
    
    /**
     * Preview merge results
     */
    public function previewMerge($primaryId, $secondaryId) {
        $primary = Capsule::table('tblclients')->where('id', $primaryId)->first();
        $secondary = Capsule::table('tblclients')->where('id', $secondaryId)->first();
        
        return [
            'primary' => [
                'id' => $primary->id,
                'email' => $primary->email,
                'name' => $primary->firstname . ' ' . $primary->lastname,
                'services' => Capsule::table('tblhosting')->where('userid', $primaryId)->count(),
                'domains' => Capsule::table('tbldomains')->where('userid', $primaryId)->count(),
                'tickets' => Capsule::table('tbltickets')->where('userid', $primaryId)->count(),
                'invoices' => Capsule::table('tblinvoices')->where('userid', $primaryId)->count()
            ],
            'secondary' => [
                'id' => $secondary->id,
                'email' => $secondary->email,
                'name' => $secondary->firstname . ' ' . $secondary->lastname,
                'services' => Capsule::table('tblhosting')->where('userid', $secondaryId)->count(),
                'domains' => Capsule::table('tbldomains')->where('userid', $secondaryId)->count(),
                'tickets' => Capsule::table('tbltickets')->where('userid', $secondaryId)->count(),
                'invoices' => Capsule::table('tblinvoices')->where('userid', $secondaryId)->count()
            ]
        ];
    }
    
    /**
     * Get merge history
     */
    public function getMergeHistory() {
        return Capsule::table('mod_merge_logs')
            ->orderBy('created_at', 'desc')
            ->get();
    }
    
    /**
     * Undo merge operation
     */
    public function undoMerge($mergeId) {
        $log = Capsule::table('mod_merge_logs')->where('id', $mergeId)->first();
        
        if (!$log) {
            throw new Exception("Merge log not found");
        }
        
        // Reverse the merge
        Capsule::table('tblhosting')
            ->where('userid', $log->primary_id)
            ->update(['userid' => $log->secondary_id]);
        
        Capsule::table('tbldomains')
            ->where('userid', $log->primary_id)
            ->update(['userid' => $log->secondary_id]);
        
        Capsule::table('tbltickets')
            ->where('userid', $log->primary_id)
            ->update(['userid' => $log->secondary_id]);
        
        // Restore secondary email
        Capsule::table('tblclients')
            ->where('id', $log->secondary_id)
            ->update([
                'email' => Capsule::raw("SUBSTRING_INDEX(email, '@', 1) || '@restored.com'"),
                'status' => 'Active'
            ]);
        
        return ['success' => true];
    }
}
```

### Step 2: Execute Merge

```php
<?php
/**
 * Execute client merge
 */
$merger = new DataMerge();

// Find duplicates
$duplicates = $merger->findDuplicates('email');

// Preview merge
$preview = $merger->previewMerge(1, 2);
print_r($preview);

// Execute merge
$result = $merger->mergeClients(1, 2, [
    'transfer_services' => true,
    'transfer_domains' => true,
    'archive_secondary' => true
]);

if ($result['success']) {
    echo "Merge completed successfully\n";
}
```

## Best Practices
- Always backup before merging
- Review merge preview carefully
- Transfer all related records
- Archive (don't delete) merged accounts
- Log all merge operations
- Consider invoice history
- Notify affected clients
- Test on staging first
- Document merge decisions
- Keep audit trail
