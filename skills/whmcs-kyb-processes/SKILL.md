# WHMCS KYB (Know Your Business) Skill

## Purpose
Provides patterns for implementing KYB business verification in WHMCS, verifying business entities, validating registration documents, and checking beneficial ownership.

## Implementation Patterns

### KYB Verification Manager
```php
<?php
class KYBVerificationManager {
    private $db;
    
    public function startVerification($clientId, $businessData) {
        $businessId = $this->createBusinessRecord($clientId, $businessData);
        
        $session = [
            'business_id' => $businessId,
            'client_id' => $clientId,
            'status' => 'in_progress',
            'checks' => json_encode(['entity', 'ownership', 'sanctions']),
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        return [
            'session_id' => $this->db->insert('mod_kyb_sessions', $session),
            'business_id' => $businessId
        ];
    }
    
    private function createBusinessRecord($clientId, $data) {
        $existing = $this->db->select("SELECT id FROM mod_kyb_businesses WHERE client_id = ?", [$clientId]);
        
        $business = [
            'client_id' => $clientId,
            'legal_name' => $data['legal_name'],
            'registration_number' => $data['registration_number'],
            'incorporation_country' => $data['incorporation_country'],
            'business_type' => $data['business_type'] ?? 'LLC',
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        if ($existing) {
            $this->db->where('client_id', $clientId)->update('mod_kyb_businesses', $business);
            return $existing['id'];
        }
        
        return $this->db->insert('mod_kyb_businesses', $business);
    }
    
    public function verifyOwnership($businessId, $owners) {
        $verified = 0;
        
        foreach ($owners as $owner) {
            $this->addBeneficialOwner($businessId, $owner);
            $verified += $owner['ownership_percentage'];
        }
        
        return [
            'total_verified' => $verified,
            'compliant' => $verified >= 100
        ];
    }
    
    private function addBeneficialOwner($businessId, $owner) {
        return $this->db->insert('mod_kyb_owners', [
            'business_id' => $businessId,
            'name' => $owner['name'],
            'ownership_percentage' => $owner['ownership_percentage'],
            'kyc_verified' => 0,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public function getStatus($businessId) {
        $business = $this->db->select("SELECT * FROM mod_kyb_businesses WHERE id = ?", [$businessId]);
        $session = $this->db->select(
            "SELECT * FROM mod_kyb_sessions WHERE business_id = ? ORDER BY created_at DESC LIMIT 1",
            [$businessId]
        );
        
        return [
            'status' => $business['verification_status'] ?? 'pending',
            'session' => $session
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_kyb_businesses (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    legal_name VARCHAR(255) NOT NULL,
    registration_number VARCHAR(100),
    incorporation_country VARCHAR(10),
    business_type VARCHAR(50),
    verification_status ENUM('pending', 'in_progress', 'verified', 'rejected') DEFAULT 'pending',
    verified_at DATETIME,
    created_at DATETIME
);

CREATE TABLE mod_kyb_sessions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    business_id INT NOT NULL,
    client_id INT NOT NULL,
    status ENUM('pending', 'in_progress', 'approved', 'rejected') DEFAULT 'pending',
    checks JSON,
    created_at DATETIME,
    completed_at DATETIME
);

CREATE TABLE mod_kyb_owners (
    id INT AUTO_INCREMENT PRIMARY KEY,
    business_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    date_of_birth DATE,
    ownership_percentage DECIMAL(5,2),
    kyc_verified TINYINT(1) DEFAULT 0,
    created_at DATETIME
);
```

## Usage Examples

### Start KYB Verification
```php
$kyb = new KYBVerificationManager();
$result = $kyb->startVerification($clientId, [
    'legal_name' => 'Acme Corp',
    'registration_number' => '12345678',
    'incorporation_country' => 'US'
]);
```

### Verify Beneficial Owners
```php
$kyb->verifyOwnership($businessId, [
    ['name' => 'John Smith', 'ownership_percentage' => 51],
    ['name' => 'Jane Doe', 'ownership_percentage' => 49]
]);
```
