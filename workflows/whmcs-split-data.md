# WHMCS Data Split Workflow

## Purpose
Split a single client account into multiple separate accounts in WHMCS.

## Prerequisites
- WHMCS installation
- Admin access
- Backup completed

## Step-by-Step Process

### Step 1: Create Data Split Handler

**Create hooks/data_split.php:**
```php
<?php
/**
 * WHMCS Data Split Handler
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class DataSplit {
    
    /**
     * Split client account
     */
    public function splitClient($sourceId, $splitConfig, $options = []) {
        $defaults = [
            'create_new_accounts' => true,
            'transfer_services' => true,
            'transfer_domains' => true,
            'keep_source_active' => false
        ];
        $options = array_merge($defaults, $options);
        
        $source = Capsule::table('tblclients')->where('id', $sourceId)->first();
        
        if (!$source) {
            throw new Exception("Source client not found");
        }
        
        // Start transaction
        Capsule::beginTransaction();
        
        try {
            $newAccounts = [];
            
            // Create new accounts for each split
            foreach ($splitConfig as $split) {
                if ($options['create_new_accounts']) {
                    $newClientId = $this->createNewClient($source, $split['profile']);
                    $newAccounts[$split['id']] = $newClientId;
                    
                    // Send welcome email to new account
                    if (!empty($split['send_welcome'])) {
                        $this->sendWelcomeEmail($newClientId, $split['profile']);
                    }
                } else {
                    $newAccounts[$split['id']] = $split['existing_id'];
                }
                
                // Transfer services
                if ($options['transfer_services'] && !empty($split['service_ids'])) {
                    $this->transferServices($split['service_ids'], $newAccounts[$split['id']]);
                }
                
                // Transfer domains
                if ($options['transfer_domains'] && !empty($split['domain_ids'])) {
                    $this->transferDomains($split['domain_ids'], $newAccounts[$split['id']]);
                }
            }
            
            // Handle source account
            if (!$options['keep_source_active']) {
                $this->archiveSourceAccount($sourceId);
            }
            
            // Log split operation
            $this->logSplitOperation($sourceId, $newAccounts, $options);
            
            Capsule::commit();
            
            return [
                'success' => true,
                'source_id' => $sourceId,
                'new_accounts' => $newAccounts
            ];
            
        } catch (Exception $e) {
            Capsule::rollBack();
            throw $e;
        }
    }
    
    /**
     * Create new client from source
     */
    private function createNewClient($source, $profile) {
        $data = [
            'uuid' => Capsule::raw('UUID()'),
            'firstname' => $profile['firstname'] ?? $source->firstname,
            'lastname' => $profile['lastname'] ?? $source->lastname,
            'email' => $profile['email'],
            'companyname' => $profile['companyname'] ?? '',
            'address1' => $profile['address1'] ?? $source->address1,
            'address2' => $profile['address2'] ?? $source->address2,
            'city' => $profile['city'] ?? $source->city,
            'state' => $profile['state'] ?? $source->state,
            'postcode' => $profile['postcode'] ?? $source->postcode,
            'country' => $profile['country'] ?? $source->country,
            'phonenumber' => $profile['phonenumber'] ?? $source->phonenumber,
            'password' => $this->generatePassword(),
            'datecreated' => Carbon::now()->toDateTimeString(),
            'language' => $source->language,
            'status' => 'Active'
        ];
        
        return Capsule::table('tblclients')->insertGetId($data);
    }
    
    /**
     * Transfer services to new account
     */
    private function transferServices($serviceIds, $newUserId) {
        Capsule::table('tblhosting')
            ->whereIn('id', $serviceIds)
            ->update([
                'userid' => $newUserId,
                'domain' => '' // Clear domain as it will be reassigned
            ]);
    }
    
    /**
     * Transfer domains to new account
     */
    private function transferDomains($domainIds, $newUserId) {
        Capsule::table('tbldomains')
            ->whereIn('id', $domainIds)
            ->update(['userid' => $newUserId]);
    }
    
    /**
     * Archive source account
     */
    private function archiveSourceAccount($clientId) {
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update([
                'email' => 'split_' . $clientId . '_' . time() . '@archived.local',
                'status' => 'Inactive',
                'notes' => Capsule::raw("CONCAT(COALESCE(notes, ''), ' [Split: " . Carbon::now()->toDateTimeString() . "]')")
            ]);
    }
    
    /**
     * Generate password
     */
    private function generatePassword($length = 12) {
        $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%';
        return substr(str_shuffle($chars), 0, $length);
    }
    
    /**
     * Send welcome email
     */
    private function sendWelcomeEmail($userId, $profile) {
        sendEmail('Welcome Email', $userId, [
            'client_password' => $profile['password'] ?? ''
        ]);
    }
    
    /**
     * Log split operation
     */
    private function logSplitOperation($sourceId, $newAccounts, $options) {
        Capsule::table('mod_split_logs')->insert([
            'source_id' => $sourceId,
            'new_accounts' => json_encode($newAccounts),
            'options' => json_encode($options),
            'performed_by' => $_SESSION['adminid'] ?? null,
            'created_at' => Carbon::now()->toDateTimeString()
        ]);
    }
    
    /**
     * Preview split results
     */
    public function previewSplit($sourceId, $splitConfig) {
        $source = Capsule::table('tblclients')->where('id', $sourceId)->first();
        
        $preview = [
            'source' => [
                'id' => $source->id,
                'email' => $source->email,
                'name' => $source->firstname . ' ' . $source->lastname
            ],
            'splits' => []
        ];
        
        foreach ($splitConfig as $split) {
            $preview['splits'][] = [
                'id' => $split['id'],
                'profile' => $split['profile'],
                'services_count' => count($split['service_ids'] ?? []),
                'domains_count' => count($split['domain_ids'] ?? [])
            ];
        }
        
        return $preview;
    }
    
    /**
     * Get split history
     */
    public function getSplitHistory() {
        return Capsule::table('mod_split_logs')
            ->orderBy('created_at', 'desc')
            ->get();
    }
}
```

### Step 2: Execute Split

```php
<?php
/**
 * Execute client split
 */
$splitter = new DataSplit();

// Preview split
$preview = $splitter->previewSplit(1, [
    [
        'id' => 1,
        'profile' => [
            'email' => 'john.business@example.com',
            'firstname' => 'John',
            'lastname' => 'Business',
            'companyname' => 'Business LLC'
        ],
        'service_ids' => [1, 2, 3],
        'domain_ids' => [1, 2]
    ],
    [
        'id' => 2,
        'profile' => [
            'email' => 'john.personal@example.com',
            'firstname' => 'John',
            'lastname' => 'Personal',
            'companyname' => ''
        ],
        'service_ids' => [4, 5],
        'domain_ids' => [3]
    ]
]);

print_r($preview);

// Execute split
$result = $splitter->splitClient(1, $preview['splits']);

if ($result['success']) {
    echo "Split completed successfully\n";
    echo "New accounts: " . json_encode($result['new_accounts']) . "\n";
}
```

## Best Practices
- Always backup before splitting
- Verify all services are assigned
- Create separate invoices if needed
- Notify affected clients
- Transfer all related data
- Consider domain ownership
- Update service credentials
- Document split reasons
- Test on staging first
- Keep audit trail
