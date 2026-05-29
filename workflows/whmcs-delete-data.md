# WHMCS Data Deletion Workflow

## Purpose
Safely delete data from WHMCS with GDPR compliance and audit trails.

## Prerequisites
- WHMCS installation
- Admin access
- Legal authorization (for GDPR)

## Step-by-Step Process

### Step 1: Create Data Deletion Handler

**Create hooks/data_deletion.php:**
```php
<?php
/**
 * WHMCS Data Deletion System
 * GDPR-compliant data removal
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class DataDeletion {
    
    private $deletionLog = [];
    
    /**
     * Delete client and all related data
     */
    public function deleteClient($clientId, $options = []) {
        $defaults = [
            'delete_services' => true,
            'delete_domains' => true,
            'delete_invoices' => false, // Usually keep for legal reasons
            'delete_tickets' => true,
            'delete_orders' => true,
            'delete_activity' => true,
            'anonymize_instead' => false, // Better for compliance
            'create_backup' => true,
            'require_confirmation' => true
        ];
        $options = array_merge($defaults, $options);
        
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        
        if (!$client) {
            throw new Exception("Client not found");
        }
        
        // Start transaction
        Capsule::beginTransaction();
        
        try {
            $deletedCounts = [];
            
            // Create backup before deletion
            if ($options['create_backup']) {
                $this->createDeletionBackup($clientId);
            }
            
            if ($options['anonymize_instead']) {
                $this->anonymizeClient($clientId);
            } else {
                // Delete related records
                if ($options['delete_services']) {
                    $deletedCounts['services'] = $this->deleteClientServices($clientId);
                }
                
                if ($options['delete_domains']) {
                    $deletedCounts['domains'] = $this->deleteClientDomains($clientId);
                }
                
                if ($options['delete_orders']) {
                    $deletedCounts['orders'] = $this->deleteClientOrders($clientId);
                }
                
                if ($options['delete_tickets']) {
                    $deletedCounts['tickets'] = $this->deleteClientTickets($clientId);
                }
                
                if ($options['delete_activity']) {
                    $deletedCounts['activity'] = $this->deleteClientActivity($clientId);
                }
                
                // Keep invoices but anonymize
                $this->anonymizeClientInvoices($clientId);
                
                // Finally delete client
                $this->deleteClient($clientId);
            }
            
            // Log deletion
            $this->logDeletion($clientId, $deletedCounts, $options);
            
            Capsule::commit();
            
            return [
                'success' => true,
                'client_id' => $clientId,
                'deleted' => $deletedCounts
            ];
            
        } catch (Exception $e) {
            Capsule::rollBack();
            throw $e;
        }
    }
    
    /**
     * Create backup before deletion
     */
    private function createDeletionBackup($clientId) {
        $backupTable = 'mod_deletion_backup';
        
        // Ensure backup table exists
        if (!Capsule::schema()->hasTable($backupTable)) {
            Capsule::schema()->create($backupTable, function($table) {
                $table->increments('id');
                $table->integer('client_id');
                $table->string('client_data', 'mediumtext');
                $table->string('related_data', 'mediumtext');
                $table->string('deleted_by');
                $table->timestamp('deleted_at');
            });
        }
        
        // Get all client data
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        $services = Capsule::table('tblhosting')->where('userid', $clientId)->get();
        $domains = Capsule::table('tbldomains')->where('userid', $clientId)->get();
        
        // Store backup
        Capsule::table($backupTable)->insert([
            'client_id' => $clientId,
            'client_data' => json_encode($client),
            'related_data' => json_encode([
                'services' => $services->toArray(),
                'domains' => $domains->toArray()
            ]),
            'deleted_by' => $_SESSION['adminid'] ?? 'system',
            'deleted_at' => Carbon::now()->toDateTimeString()
        ]);
    }
    
    /**
     * Anonymize client data (GDPR)
     */
    private function anonymizeClient($clientId) {
        $anonymizedData = [
            'firstname' => 'Deleted',
            'lastname' => 'User',
            'email' => 'deleted_' . $clientId . '_' . time() . '@anonymized.local',
            'companyname' => '',
            'address1' => '',
            'address2' => '',
            'city' => '',
            'state' => '',
            'postcode' => '',
            'country' => '',
            'phonenumber' => '',
            'notes' => Capsule::raw("CONCAT(COALESCE(notes, ''), ' [Anonymized: " . Carbon::now()->toDateTimeString() . "]')"),
            'status' => 'Closed'
        ];
        
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update($anonymizedData);
    }
    
    /**
     * Delete client services
     */
    private function deleteClientServices($clientId) {
        return Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->delete();
    }
    
    /**
     * Delete client domains
     */
    private function deleteClientDomains($clientId) {
        return Capsule::table('tbldomains')
            ->where('userid', $clientId)
            ->delete();
    }
    
    /**
     * Delete client orders
     */
    private function deleteClientOrders($clientId) {
        // Get order IDs first
        $orderIds = Capsule::table('tblorders')
            ->where('userid', $clientId)
            ->pluck('id')
            ->toArray();
        
        // Delete order items
        if (!empty($orderIds)) {
            Capsule::table('tblorderitems')
                ->whereIn('orderid', $orderIds)
                ->delete();
        }
        
        // Delete orders
        return Capsule::table('tblorders')
            ->where('userid', $clientId)
            ->delete();
    }
    
    /**
     * Delete client tickets
     */
    private function deleteClientTickets($clientId) {
        // Get ticket IDs
        $ticketIds = Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->pluck('id')
            ->toArray();
        
        // Delete ticket replies
        if (!empty($ticketIds)) {
            Capsule::table('tblticketreplies')
                ->whereIn('tid', $ticketIds)
                ->delete();
            
            // Delete attachments
            Capsule::table('tblticketattachments')
                ->whereIn('tid', $ticketIds)
                ->delete();
        }
        
        // Delete tickets
        return Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->delete();
    }
    
    /**
     * Delete client activity
     */
    private function deleteClientActivity($clientId) {
        return Capsule::table('tblactivitylog')
            ->where('userid', $clientId)
            ->delete();
    }
    
    /**
     * Anonymize client invoices (keep for legal)
     */
    private function anonymizeClientInvoices($clientId) {
        Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->update([
                'userid' => 0,
                'notes' => Capsule::raw("CONCAT(COALESCE(notes, ''), ' [Client deleted: " . Carbon::now()->toDateTimeString() . "]')")
            ]);
    }
    
    /**
     * Delete client (final step)
     */
    private function deleteClient($clientId) {
        return Capsule::table('tblclients')
            ->where('id', $clientId)
            ->delete();
    }
    
    /**
     * Log deletion
     */
    private function logDeletion($clientId, $counts, $options) {
        Capsule::table('mod_deletion_logs')->insert([
            'client_id' => $clientId,
            'deleted_counts' => json_encode($counts),
            'options' => json_encode($options),
            'deleted_by' => $_SESSION['adminid'] ?? 'system',
            'created_at' => Carbon::now()->toDateTimeString()
        ]);
    }
    
    /**
     * Get deletion logs
     */
    public function getDeletionLogs($limit = 50) {
        return Capsule::table('mod_deletion_logs')
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
    }
    
    /**
     * Preview deletion
     */
    public function previewDeletion($clientId) {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        
        return [
            'client' => [
                'id' => $client->id,
                'email' => $client->email,
                'name' => $client->firstname . ' ' . $client->lastname,
                'created' => $client->datecreated
            ],
            'related_counts' => [
                'services' => Capsule::table('tblhosting')->where('userid', $clientId)->count(),
                'domains' => Capsule::table('tbldomains')->where('userid', $clientId)->count(),
                'invoices' => Capsule::table('tblinvoices')->where('userid', $clientId)->count(),
                'tickets' => Capsule::table('tbltickets')->where('userid', $clientId)->count(),
                'orders' => Capsule::table('tblorders')->where('userid', $clientId)->count()
            ]
        ];
    }
}
```

### Step 2: Execute Deletion

```php
<?php
/**
 * Execute client deletion
 */
$deletion = new DataDeletion();

// Preview first
$preview = $deletion->previewDeletion(123);
print_r($preview);

// Execute deletion
$result = $deletion->deleteClient(123, [
    'delete_services' => true,
    'delete_domains' => true,
    'delete_orders' => true,
    'anonymize_invoices' => true,
    'create_backup' => true
]);

if ($result['success']) {
    echo "Client deleted successfully.\n";
    print_r($result['deleted']);
}
```

## Best Practices
- Always create backup first
- Consider anonymization instead
- Keep invoices for legal compliance
- Log all deletions
- Require authorization
- Test on staging first
- Document deletion reasons
- Follow GDPR guidelines
- Maintain audit trail
- Verify complete deletion
