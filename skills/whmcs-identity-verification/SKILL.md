# WHMCS Identity Verification Skill

## Purpose
Provides patterns for implementing KYC identity verification in WHMCS, collecting and validating identity documents, verifying personal information, and managing verification workflows.

## Implementation Patterns

### KYC Verification Manager
```php
<?php
class KYCVerificationManager {
    private $db;
    
    public function __construct() {
        $this->db = console::db();
    }
    
    public function startVerification($clientId, $provider = 'jumio') {
        $session = [
            'client_id' => $clientId,
            'provider' => $provider,
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s'),
            'expires_at' => date('Y-m-d H:i:s', strtotime('+7 days'))
        ];
        
        $sessionId = $this->db->insert('mod_kyc_sessions', $session);
        
        return ['session_id' => $sessionId];
    }
    
    public function getStatus($clientId) {
        $session = $this->db->select(
            "SELECT * FROM mod_kyc_sessions WHERE client_id = ? ORDER BY created_at DESC LIMIT 1",
            [$clientId]
        );
        
        if (!$session) {
            return ['status' => 'not_started'];
        }
        
        return [
            'status' => $session['status'],
            'session_id' => $session['id']
        ];
    }
    
    public function isClientVerified($clientId) {
        $client = $this->db->select("SELECT kyc_status FROM tblclients WHERE id = ?", [$clientId]);
        return ($client['kyc_status'] ?? '') === 'verified';
    }
    
    public function handleWebhook($provider, $payload) {
        $sessionId = $payload['reference_id'] ?? null;
        $status = $payload['status'] ?? null;
        
        if ($status === 'approved') {
            $this->approveVerification($sessionId);
        } elseif ($status === 'rejected') {
            $this->rejectVerification($sessionId, $payload['reasons'] ?? []);
        }
        
        return true;
    }
    
    private function approveVerification($sessionId) {
        $session = $this->db->select("SELECT client_id FROM mod_kyc_sessions WHERE id = ?", [$sessionId]);
        
        $this->db->where('id', $sessionId)->update('mod_kyc_sessions', [
            'status' => 'approved',
            'completed_at' => date('Y-m-d H:i:s')
        ]);
        
        $this->db->where('id', $session['client_id'])->update('tblclients', [
            'kyc_status' => 'verified',
            'kyc_verified_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_kyc_sessions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    provider VARCHAR(50),
    status ENUM('pending', 'in_progress', 'approved', 'rejected', 'expired') DEFAULT 'pending',
    documents JSON,
    created_at DATETIME,
    completed_at DATETIME,
    expires_at DATETIME,
    INDEX idx_client (client_id)
);

CREATE TABLE mod_kyc_documents (
    id INT AUTO_INCREMENT PRIMARY KEY,
    session_id INT NOT NULL,
    document_type VARCHAR(50),
    file_path VARCHAR(500),
    validated TINYINT(1) DEFAULT 0,
    uploaded_at DATETIME
);
```

## Usage Examples

### Start Verification
```php
$kyc = new KYCVerificationManager();
if (!$kyc->isClientVerified($clientId)) {
    $result = $kyc->startVerification($clientId);
    // Redirect to verification
}
```

### Handle Webhook
```php
$kyc->handleWebhook('jumio', $_POST);
```
