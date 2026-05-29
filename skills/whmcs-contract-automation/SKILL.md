# WHMCS Contract Automation Skill

## Purpose
Provides patterns for implementing contract automation in WHMCS, managing contract lifecycle, generating contract documents, tracking obligations, and automating renewals.

## Implementation Patterns

### Contract Manager
```php
<?php
// includes/ContractAutomation.class.php

class ContractManager {
    private $db;
    private $templates;
    
    public function __construct() {
        $this->db = console::db();
        $this->loadTemplates();
    }
    
    private function loadTemplates() {
        $this->templates = $this->db->select(
            "SELECT * FROM mod_contract_templates WHERE is_active = 1"
        );
    }
    
    // Create a new contract
    public function createContract($data) {
        $contract = [
            'contract_number' => $this->generateContractNumber(),
            'client_id' => $data['client_id'],
            'template_id' => $data['template_id'],
            'title' => $data['title'],
            'type' => $data['type'] ?? 'standard',
            'status' => 'draft',
            'effective_date' => $data['effective_date'],
            'expiration_date' => $data['expiration_date'],
            'auto_renew' => $data['auto_renew'] ?? true,
            'renewal_terms' => json_encode($data['renewal_terms'] ?? []),
            'terms' => $data['terms'] ?? '',
            'value' => $data['value'] ?? 0,
            'currency' => $data['currency'] ?? 'USD',
            'signatures' => json_encode([]),
            'attachments' => json_encode([]),
            'metadata' => json_encode($data['metadata'] ?? []),
            'created_by' => $this->getCurrentUserId(),
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s')
        ];
        
        $contractId = $this->db->insert('mod_contracts', $contract);
        
        // Create initial version
        $this->createVersion($contractId, $contract['terms'], 'Initial version');
        
        // Initialize obligations
        $this->initializeObligations($contractId);
        
        return $contractId;
    }
    
    private function generateContractNumber() {
        $prefix = 'CTR';
        $year = date('Y');
        $month = date('m');
        
        $lastNumber = $this->db->select(
            "SELECT contract_number FROM mod_contracts 
             WHERE contract_number LIKE ? 
             ORDER BY id DESC LIMIT 1",
            ["{$prefix}-{$year}{$month}%"]
        );
        
        if ($lastNumber) {
            $sequence = (int)substr($lastNumber['contract_number'], -4) + 1;
        } else {
            $sequence = 1;
        }
        
        return sprintf("%s-%s%s-%04d", $prefix, $year, $month, $sequence);
    }
    
    // Generate contract document
    public function generateDocument($contractId, $format = 'pdf') {
        $contract = $this->getContract($contractId);
        $template = $this->getTemplate($contract['template_id']);
        
        $content = $template['content'];
        
        // Replace placeholders with actual values
        $content = $this->replacePlaceholders($content, $contract);
        
        // Apply formatting
        $document = $this->formatDocument($content, $format);
        
        // Store generated document
        $this->saveDocument($contractId, $document, $format);
        
        return $document;
    }
    
    private function replacePlaceholders($content, $contract) {
        $client = $this->getClient($contract['client_id']);
        
        $replacements = [
            '{{contract_number}}' => $contract['contract_number'],
            '{{client_name}}' => $client['companyname'] ?? $client['firstname'] . ' ' . $client['lastname'],
            '{{client_address}}' => $client['address1'] . "\n" . $client['city'] . ', ' . $client['state'] . ' ' . $client['postcode'],
            '{{effective_date}}' => date('F j, Y', strtotime($contract['effective_date'])),
            '{{expiration_date}}' => date('F j, Y', strtotime($contract['expiration_date'])),
            '{{contract_value}}' => number_format($contract['value'], 2),
            '{{currency}}' => $contract['currency'],
            '{{signing_date}}' => date('F j, Y'),
            '{{terms}}' => $contract['terms'],
            '{{auto_renewal}}' => $contract['auto_renew'] ? 'Yes' : 'No',
            '{{renewal_terms}}' => $this->formatRenewalTerms($contract['renewal_terms']),
            '{{company_name}}' => get_config('CompanyName'),
            '{{company_address}}' => get_config('Address1') . "\n" . get_config('City') . ', ' . get_config('State') . ' ' . get_config('Postcode')
        ];
        
        return str_replace(array_keys($replacements), array_values($replacements), $content);
    }
    
    // Sign a contract
    public function signContract($contractId, $signerId, $signatureData, $type = 'digital') {
        $contract = $this->getContract($contractId);
        
        if ($contract['status'] === 'expired') {
            throw new Exception("Cannot sign expired contract");
        }
        
        $signature = [
            'signer_id' => $signerId,
            'signer_type' => $this->getSignerType($signerId),
            'type' => $type,
            'signature' => $signatureData,
            'signed_at' => date('Y-m-d H:i:s'),
            'ip_address' => $this->getClientIP(),
            'user_agent' => $this->getUserAgent()
        ];
        
        $signatures = json_decode($contract['signatures'], true);
        $signatures[] = $signature;
        
        // Update contract
        $this->db->where('id', $contractId)
            ->update('mod_contracts', [
                'signatures' => json_encode($signatures),
                'status' => $this->determineStatus($signatures),
                'signed_at' => count($signatures) === 2 ? date('Y-m-d H:i:s') : null,
                'updated_at' => date('Y-m-d H:i:s')
            ]);
        
        // Create audit entry
        $this->logSigning($contractId, $signature);
        
        // Check if fully signed
        if (count($signatures) === 2) {
            $this->activateContract($contractId);
        }
        
        return $signature;
    }
    
    private function determineStatus($signatures) {
        $requiredSigners = 2;
        
        if (count($signatures) === $requiredSigners) {
            return 'signed';
        }
        
        return 'pending_signature';
    }
    
    // Manage contract renewal
    public function processRenewals() {
        $expiringContracts = $this->getExpiringContracts(30);
        
        foreach ($expiringContracts as $contract) {
            if ($contract['auto_renew']) {
                $this->processAutoRenewal($contract);
            } else {
                $this->sendExpirationNotice($contract);
            }
        }
    }
    
    private function processAutoRenewal($contract) {
        $renewalTerms = json_decode($contract['renewal_terms'], true);
        
        // Check if all conditions for renewal are met
        if (!$this->canRenew($contract)) {
            $this->flagRenewalIssue($contract);
            return false;
        }
        
        // Create renewal proposal
        $renewal = [
            'original_contract_id' => $contract['id'],
            'new_effective_date' => date('Y-m-d', strtotime($contract['expiration_date'] . '+1 day')),
            'new_expiration_date' => date('Y-m-d', strtotime($contract['expiration_date'] . '+1 year')),
            'renewal_value' => $this->calculateRenewalValue($contract, $renewalTerms),
            'status' => 'pending_approval',
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        $renewalId = $this->db->insert('mod_contract_renewals', $renewal);
        
        // Notify client
        $this->sendRenewalNotification($contract, $renewal);
        
        return $renewalId;
    }
    
    private function calculateRenewalValue($contract, $terms) {
        $baseValue = $contract['value'];
        $adjustmentType = $terms['adjustment_type'] ?? 'fixed';
        $adjustmentValue = $terms['adjustment_value'] ?? 0;
        
        switch ($adjustmentType) {
            case 'percentage':
                return $baseValue * (1 + $adjustmentValue / 100);
            case 'fixed':
                return $baseValue + $adjustmentValue;
            case 'inflation':
                return $baseValue * (1 + 0.03); // 3% inflation default
            default:
                return $baseValue;
        }
    }
    
    // Track contract obligations
    public function trackObligations($contractId) {
        $obligations = $this->getObligations($contractId);
        
        foreach ($obligations as &$obligation) {
            // Check if deadline passed
            if ($obligation['deadline'] < date('Y-m-d') && $obligation['status'] === 'pending') {
                $obligation['status'] = 'overdue';
                $this->updateObligation($obligation);
                
                // Notify
                $this->notifyObligationOverdue($obligation);
            }
            
            // Check upcoming deadlines
            $daysUntil = (strtotime($obligation['deadline']) - time()) / 86400;
            if ($daysUntil <= 7 && $daysUntil > 0 && $obligation['status'] === 'pending') {
                $this->sendDeadlineWarning($obligation);
            }
        }
        
        return $obligations;
    }
    
    private function initializeObligations($contractId) {
        $template = $this->getContractTemplate($contractId);
        $obligationTemplates = json_decode($template['obligations'], true) ?? [];
        
        foreach ($obligationTemplates as $obligation) {
            $this->createObligation($contractId, $obligation);
        }
    }
    
    private function createObligation($contractId, $data) {
        $obligation = [
            'contract_id' => $contractId,
            'title' => $data['title'],
            'description' => $data['description'],
            'type' => $data['type'],
            'deadline' => $data['deadline'],
            'assignee_type' => $data['assignee_type'] ?? 'client',
            'assignee_id' => $data['assignee_id'] ?? null,
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        return $this->db->insert('mod_contract_obligations', $obligation);
    }
}
```

### Contract Templates
```php
class ContractTemplates {
    public static function getStandardTemplate() {
        return [
            'name' => 'Standard Service Agreement',
            'type' => 'service',
            'content' => '
{{company_name}}
{{company_address}}

SERVICE AGREEMENT

Contract Number: {{contract_number}}
Effective Date: {{effective_date}}
Expiration Date: {{expiration_date}}

PARTIES:
Provider: {{company_name}}
Client: {{client_name}}

ADDRESS:
{{client_address}}

1. SERVICES
The Provider agrees to provide services as specified in the attached service schedule.

2. TERM
This Agreement shall commence on {{effective_date}} and continue until {{expiration_date}}.
{{auto_renewal}} Auto-renewal: {{auto_renewal}}.

3. PAYMENT
The total contract value is {{currency}} {{contract_value}}.

4. OBLIGATIONS
[Detailed obligations section]

5. TERMINATION
Either party may terminate with 30 days written notice.

6. GOVERNING LAW
This Agreement shall be governed by applicable law.

SIGNATURES:

_____________________________
{{company_name}}
Date: {{signing_date}}

_____________________________
{{client_name}}
Date: {{signing_date}}
            ',
            'obligations' => json_encode([
                [
                    'title' => 'Payment Due',
                    'type' => 'payment',
                    'deadline' => 'monthly',
                    'assignee_type' => 'client'
                ],
                [
                    'title' => 'Service Review',
                    'type' => 'review',
                    'deadline' => 'quarterly',
                    'assignee_type' => 'provider'
                ]
            ]),
            'is_active' => 1
        ];
    }
    
    public static function getNDA() {
        return [
            'name' => 'Non-Disclosure Agreement',
            'type' => 'legal',
            'content' => '
NON-DISCLOSURE AGREEMENT

This NDA is entered into as of {{effective_date}} between {{company_name}} and {{client_name}}.

1. CONFIDENTIAL INFORMATION
Both parties agree to protect confidential information shared during business dealings.

2. OBLIGATIONS
- Maintain confidentiality for {{expiration_date}}
- Not disclose to third parties without consent
- Use only for intended purpose

3. RETURN OF INFORMATION
Upon termination, all confidential materials must be returned or destroyed.

4. TERM
This NDA shall remain in effect until {{expiration_date}}.

Agreed and signed on {{signing_date}}.
            ',
            'obligations' => '[]',
            'is_active' => 1
        ];
    }
}
```

### Contract Compliance Tracking
```php
class ContractComplianceTracker {
    public function checkCompliance($contractId) {
        $contract = $this->getContract($contractId);
        $checks = [];
        
        // Check signature status
        $checks['signatures'] = $this->checkSignatures($contract);
        
        // Check obligations
        $checks['obligations'] = $this->checkObligations($contractId);
        
        // Check payment status
        $checks['payments'] = $this->checkPaymentCompliance($contract);
        
        // Check renewal status
        $checks['renewal'] = $this->checkRenewalCompliance($contract);
        
        // Check amendments
        $checks['amendments'] = $this->checkAmendments($contractId);
        
        return [
            'contract_id' => $contractId,
            'compliant' => $this->isFullyCompliant($checks),
            'checks' => $checks,
            'issues' => $this->identifyIssues($checks),
            'checked_at' => date('Y-m-d H:i:s')
        ];
    }
    
    private function checkSignatures($contract) {
        $signatures = json_decode($contract['signatures'], true);
        
        return [
            'required' => 2,
            'obtained' => count($signatures),
            'compliant' => count($signatures) === 2
        ];
    }
    
    private function checkPaymentCompliance($contract) {
        $outstandingInvoices = $this->db->select(
            "SELECT COUNT(*) as count, SUM(total) as amount
             FROM tblinvoices
             WHERE userid = ? AND status IN ('Draft', 'Unpaid', 'Overdue')
             AND duedate < NOW()",
            [$contract['client_id']]
        );
        
        return [
            'has_overdue' => $outstandingInvoices['count'] > 0,
            'overdue_amount' => $outstandingInvoices['amount'] ?? 0,
            'compliant' => $outstandingInvoices['count'] == 0
        ];
    }
}
```

### Contract Amendment Management
```php
class ContractAmendmentManager {
    public function createAmendment($contractId, $data) {
        $amendment = [
            'contract_id' => $contractId,
            'amendment_number' => $this->generateAmendmentNumber($contractId),
            'title' => $data['title'],
            'description' => $data['description'],
            'changes' => json_encode($data['changes']),
            'effective_date' => $data['effective_date'],
            'status' => 'pending',
            'created_by' => $this->getCurrentUserId(),
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        return $this->db->insert('mod_contract_amendments', $amendment);
    }
    
    public function approveAmendment($amendmentId, $signerId, $signature) {
        $this->db->where('id', $amendmentId)
            ->update('mod_contract_amendments', [
                'status' => 'approved',
                'approved_by' => $signerId,
                'signature' => $signature,
                'approved_at' => date('Y-m-d H:i:s')
            ]);
        
        // Apply changes to contract
        $this->applyAmendment($amendmentId);
        
        return true;
    }
    
    private function applyAmendment($amendmentId) {
        $amendment = $this->getAmendment($amendmentId);
        $changes = json_decode($amendment['changes'], true);
        
        // Apply each change
        foreach ($changes as $change) {
            $this->applyChange($amendment['contract_id'], $change);
        }
        
        // Create new version
        $this->createVersion(
            $amendment['contract_id'],
            $this->getContractTerms($amendment['contract_id']),
            "Amendment {$amendment['amendment_number']}"
        );
    }
    
    private function applyChange($contractId, $change) {
        $field = $change['field'];
        $value = $change['value'];
        
        $this->db->where('id', $contractId)
            ->update('mod_contracts', [
                $field => $value,
                'updated_at' => date('Y-m-d H:i:s')
            ]);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_contracts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    contract_number VARCHAR(50) UNIQUE NOT NULL,
    client_id INT NOT NULL,
    template_id INT,
    title VARCHAR(255) NOT NULL,
    type VARCHAR(50),
    status ENUM('draft', 'pending_signature', 'signed', 'active', 'expired', 'terminated', 'amended') DEFAULT 'draft',
    effective_date DATE NOT NULL,
    expiration_date DATE NOT NULL,
    auto_renew TINYINT(1) DEFAULT 1,
    renewal_terms JSON,
    terms TEXT,
    value DECIMAL(15,2),
    currency VARCHAR(10) DEFAULT 'USD',
    signatures JSON,
    attachments JSON,
    metadata JSON,
    signed_at DATETIME,
    created_by INT,
    created_at DATETIME,
    updated_at DATETIME,
    INDEX idx_client (client_id),
    INDEX idx_status (status),
    INDEX idx_expiration (expiration_date)
);

CREATE TABLE mod_contract_templates (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50),
    content TEXT,
    obligations JSON,
    is_active TINYINT(1) DEFAULT 1,
    created_at DATETIME
);

CREATE TABLE mod_contract_versions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    contract_id INT NOT NULL,
    version_number INT NOT NULL,
    content TEXT,
    change_summary VARCHAR(255),
    created_by INT,
    created_at DATETIME,
    INDEX idx_contract (contract_id)
);

CREATE TABLE mod_contract_obligations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    contract_id INT NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    type VARCHAR(50),
    deadline DATE,
    assignee_type VARCHAR(20),
    assignee_id INT,
    status ENUM('pending', 'completed', 'overdue', 'waived') DEFAULT 'pending',
    completed_at DATETIME,
    completed_by INT,
    created_at DATETIME,
    INDEX idx_contract (contract_id),
    INDEX idx_status (status)
);

CREATE TABLE mod_contract_renewals (
    id INT AUTO_INCREMENT PRIMARY KEY,
    original_contract_id INT NOT NULL,
    new_effective_date DATE,
    new_expiration_date DATE,
    renewal_value DECIMAL(15,2),
    status ENUM('pending_approval', 'approved', 'rejected', 'completed') DEFAULT 'pending_approval',
    approved_by INT,
    approved_at DATETIME,
    created_at DATETIME
);

CREATE TABLE mod_contract_amendments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    contract_id INT NOT NULL,
    amendment_number VARCHAR(50) NOT NULL,
    title VARCHAR(255),
    description TEXT,
    changes JSON,
    effective_date DATE,
    status ENUM('draft', 'pending', 'approved', 'rejected') DEFAULT 'draft',
    approved_by INT,
    signature TEXT,
    approved_at DATETIME,
    created_by INT,
    created_at DATETIME,
    INDEX idx_contract (contract_id)
);

CREATE TABLE mod_contract_signings (
    id INT AUTO_INCREMENT PRIMARY KEY,
    contract_id INT NOT NULL,
    signer_id INT NOT NULL,
    signer_type VARCHAR(20),
    signature_type ENUM('digital', 'wet', 'clickwrap') DEFAULT 'digital',
    signature_data TEXT,
    ip_address VARCHAR(45),
    user_agent VARCHAR(500),
    signed_at DATETIME,
    INDEX idx_contract (contract_id)
);
```

## Usage Examples

### Create a New Contract
```php
$manager = new ContractManager();
$contractId = $manager->createContract([
    'client_id' => $clientId,
    'template_id' => $templateId,
    'title' => 'Enterprise Hosting Agreement',
    'effective_date' => '2026-01-01',
    'expiration_date' => '2027-01-01',
    'auto_renew' => true,
    'value' => 12000,
    'renewal_terms' => [
        'adjustment_type' => 'percentage',
        'adjustment_value' => 5
    ]
]);
```

### Generate Contract Document
```php
$manager = new ContractManager();
$document = $manager->generateDocument($contractId, 'pdf');

// Save or send document to client
```

### Process Contract Renewals (Cron Job)
```php
$manager = new ContractManager();
$manager->processRenewals();
```

### Track Contract Obligations
```php
$tracker = new ContractComplianceTracker();
$compliance = $tracker->checkCompliance($contractId);

if (!$compliance['compliant']) {
    foreach ($compliance['issues'] as $issue) {
        echo "Issue: {$issue['description']}\n";
    }
}
```

## Integration Points

- WHMCS client management for contract associations
- WHMCS invoices for payment tracking
- Email system for notifications
- Document management for contract storage
- E-signature providers for signing workflows
- CRM integrations for contract sync