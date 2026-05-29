# WHMCS Data Anonymization Workflow

## Purpose
Anonymize personal data in WHMCS for GDPR compliance and testing.

## Prerequisites
- WHMCS installation
- Admin access

## Step-by-Step Process

### Step 1: Create Data Anonymization Handler

**Create hooks/data_anonymize.php:**
```php
<?php
/**
 * WHMCS Data Anonymization System
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class DataAnonymization {
    
    /**
     * Anonymize client data
     */
    public function anonymizeClient($clientId, $options = []) {
        $defaults = [
            'preserve_email_domain' => false,
            'preserve_invoices' => true,
            'anonymize_services' => true
        ];
        $options = array_merge($defaults, $options);
        
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        
        if (!$client) {
            throw new Exception("Client not found");
        }
        
        $anonymizedData = [
            'firstname' => 'User' . substr(md5($client->id), 0, 4),
            'lastname' => 'Account',
            'companyname' => '',
            'address1' => '',
            'address2' => '',
            'city' => '',
            'state' => '',
            'postcode' => '',
            'country' => 'XX',
            'phonenumber' => '',
            'notes' => Capsule::raw("CONCAT(COALESCE(notes, ''), ' [Anonymized: " . Carbon::now()->toDateTimeString() . "]')")
        ];
        
        // Generate anonymized email
        if ($options['preserve_email_domain']) {
            $domain = substr(strrchr($client->email, '@'), 1);
            $anonymizedData['email'] = 'anon_' . $client->id . '@' . $domain;
        } else {
            $anonymizedData['email'] = 'anon_' . $client->id . '_' . time() . '@anonymized.local';
        }
        
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update($anonymizedData);
        
        // Anonymize services
        if ($options['anonymize_services']) {
            $this->anonymizeClientServices($clientId);
        }
        
        return [
            'success' => true,
            'client_id' => $clientId,
            'anonymized_at' => Carbon::now()->toDateTimeString()
        ];
    }
    
    /**
     * Anonymize client services
     */
    private function anonymizeClientServices($clientId) {
        Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->update([
                'domain' => Capsule::raw("CONCAT('deleted-', id, '.local')"),
                'username' => Capsule::raw("CONCAT('deleted_', id)")
            ]);
    }
    
    /**
     * Batch anonymize old clients
     */
    public function batchAnonymize($beforeDate, $options = []) {
        $clients = Capsule::table('tblclients')
            ->where('datecreated', '<', $beforeDate)
            ->where('status', '!=', 'Active')
            ->get(['id']);
        
        $anonymized = 0;
        
        foreach ($clients as $client) {
            try {
                $this->anonymizeClient($client->id, $options);
                $anonymized++;
            } catch (Exception $e) {
                // Continue with others
            }
        }
        
        return [
            'anonymized' => $anonymized,
            'before_date' => $beforeDate
        ];
    }
}
```

### Step 2: Execute Anonymization

```php
<?php
$anonymizer = new DataAnonymization();

// Anonymize specific client
$result = $anonymizer->anonymizeClient(123);

// Batch anonymize
$result = $anonymizer->batchAnonymize('2023-01-01');
```

## Best Practices
- Document all anonymization
- Test on sample data
- Maintain referential integrity
- Follow GDPR guidelines
- Keep audit trail
