# WHMCS E-Sign Integration Skill

## Purpose
Provides patterns for integrating electronic signature services into WHMCS, enabling legally binding digital signatures for contracts, agreements, and documents.

## Implementation Patterns

### E-Sign Integration Base
```php
<?php
// includes/ESignIntegration.class.php

abstract class ESignProvider {
    protected $config;
    protected $apiBaseUrl;
    protected $apiKey;
    
    abstract public function createEnvelope($document, $signers);
    abstract public function getEnvelopeStatus($envelopeId);
    abstract public function getSignedDocument($envelopeId);
    abstract public function voidEnvelope($envelopeId, $reason);
    abstract public function getSignerUrl($envelopeId, $signerEmail);
}

class DocuSignIntegration extends ESignProvider {
    public function __construct() {
        $this->config = $this->loadConfig();
        $this->apiBaseUrl = $this->config['environment'] === 'production' 
            ? 'https://na4.docusign.net/restapi' 
            : 'https://demo.docusign.net/restapi';
        $this->apiKey = $this->config['api_key'];
    }
    
    public function createEnvelope($document, $signers) {
        $payload = [
            'emailSubject' => $document['subject'],
            'emailBlurb' => $document['message'] ?? '',
            'documents' => [$this->prepareDocument($document)],
            'recipients' => $this->prepareRecipients($signers),
            'status' => 'sent'
        ];
        
        $response = $this->makeRequest(
            '/v2.1/accounts/' . $this->config['account_id'] . '/envelopes',
            'POST',
            $payload
        );
        
        $this->logEnvelopeCreation($response, $document, $signers);
        
        return [
            'envelope_id' => $response['envelopeId'],
            'status' => $response['status'],
            'sent_at' => date('Y-m-d H:i:s')
        ];
    }
    
    public function getEnvelopeStatus($envelopeId) {
        $response = $this->makeRequest(
            "/v2.1/accounts/{$this->config['account_id']}/envelopes/{$envelopeId}"
        );
        
        return [
            'envelope_id' => $envelopeId,
            'status' => $response['status'],
            'sent_at' => $response['sentDateTime'],
            'delivered_at' => $response['deliveredDateTime'] ?? null,
            'completed_at' => $response['completedDateTime'] ?? null,
            'voided_at' => $response['voidedDateTime'] ?? null
        ];
    }
    
    public function getSignedDocument($envelopeId) {
        $response = $this->makeRequest(
            "/v2.1/accounts/{$this->config['account_id']}/envelopes/{$envelopeId}/documents/combined"
        );
        
        return [
            'document' => base64_encode($response),
            'content_type' => 'application/pdf'
        ];
    }
    
    private function prepareDocument($document) {
        return [
            'documentBase64' => base64_encode($document['content']),
            'name' => $document['name'],
            'fileExtension' => 'pdf',
            'documentId' => '1'
        ];
    }
    
    private function prepareRecipients($signers) {
        $recipients = ['signers' => []];
        
        foreach ($signers as $index => $signer) {
            $recipients['signers'][] = [
                'email' => $signer['email'],
                'name' => $signer['name'],
                'recipientId' => (string)($index + 1),
                'routingOrder' => (string)$signer['order'],
                'tabs' => $this->generateTabs($signer['signature_fields'])
            ];
        }
        
        return $recipients;
    }
    
    private function generateTabs($fields) {
        $tabs = ['signHereTabs' => []];
        
        foreach ($fields as $field) {
            $tabs['signHereTabs'][] = [
                'documentId' => '1',
                'pageNumber' => $field['page'],
                'xPosition' => $field['x'],
                'yPosition' => $field['y']
            ];
        }
        
        return $tabs;
    }
    
    private function makeRequest($endpoint, $method = 'GET', $data = null) {
        $ch = curl_init($this->apiBaseUrl . $endpoint);
        
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->getAccessToken(),
                'Content-Type: application/json'
            ],
            CURLOPT_CUSTOMREQUEST => $method
        ]);
        
        if ($data) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode >= 400) {
            throw new Exception("DocuSign API error: " . $response);
        }
        
        return json_decode($response, true);
    }
    
    private function getAccessToken() {
        // Implementation for OAuth token retrieval
        return $this->apiKey;
    }
}

class HelloSignIntegration extends ESignProvider {
    public function __construct() {
        $this->config = $this->loadConfig();
    }
    
    public function createEnvelope($document, $signers) {
        $payload = [
            'title' => $document['subject'],
            'subject' => $document['subject'],
            'message' => $document['message'] ?? '',
            'file_url' => $document['file_url'],
            'signers' => array_map(function($s) {
                return [
                    'email_address' => $s['email'],
                    'name' => $s['name'],
                    'order' => $s['order']
                ];
            }, $signers),
            'signing_options' => [
                'draw' => true,
                'type' => true,
                'upload' => true,
                'phone' => false
            ],
            'metadata' => [
                'whmcs_document_id' => $document['id'] ?? null
            ]
        ];
        
        $response = $this->apiRequest('/signature_requests/send', $payload);
        
        return [
            'envelope_id' => $response['signature_request']['signature_request_id'],
            'sign_url' => $response['signature_request']['sign_url'],
            'status' => 'sent'
        ];
    }
    
    private function apiRequest($endpoint, $data = null) {
        $ch = curl_init('https://api.hellosign.com/v3' . $endpoint);
        
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => $this->config['api_key'] . ':',
            CURLOPT_HTTPHEADER => ['Content-Type: application/json']
        ]);
        
        if ($data) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
}
```

### WHMCS E-Sign Manager
```php
class WHMCSEsignManager {
    private $db;
    private $provider;
    
    public function __construct($provider = 'docusign') {
        $this->db = console::db();
        $this->provider = $this->initializeProvider($provider);
    }
    
    private function initializeProvider($provider) {
        switch ($provider) {
            case 'docusign':
                return new DocuSignIntegration();
            case 'hellosign':
                return new HelloSignIntegration();
            default:
                throw new Exception("Unknown e-sign provider: {$provider}");
        }
    }
    
    // Send document for signature
    public function sendForSignature($documentId, $signerData, $options = []) {
        $document = $this->getDocument($documentId);
        $signers = $this->prepareSigners($signerData, $document);
        
        $envelope = $this->provider->createEnvelope($document, $signers);
        
        $signatureRequest = [
            'document_id' => $documentId,
            'envelope_id' => $envelope['envelope_id'],
            'provider' => get_class($this->provider),
            'status' => 'sent',
            'sent_at' => date('Y-m-d H:i:s'),
            'signers' => json_encode($signers),
            'options' => json_encode($options)
        ];
        
        $requestId = $this->db->insert('mod_esign_requests', $signatureRequest);
        
        // Send email notification
        $this->notifySigners($requestId, $signers);
        
        return $requestId;
    }
    
    private function prepareSigners($signerData, $document) {
        $signers = [];
        
        foreach ($signerData as $index => $signer) {
            $signers[] = [
                'email' => $signer['email'],
                'name' => $signer['name'],
                'order' => $signer['order'] ?? $index + 1,
                'signature_fields' => $signer['signature_fields'] ?? [
                    ['page' => 1, 'x' => 100, 'y' => 700]
                ]
            ];
        }
        
        return $signers;
    }
    
    // Check signature status
    public function checkStatus($requestId) {
        $request = $this->getRequest($requestId);
        
        $status = $this->provider->getEnvelopeStatus($request['envelope_id']);
        
        // Update local status
        if ($status['status'] !== $request['status']) {
            $this->updateRequestStatus($requestId, $status['status']);
        }
        
        // Check if fully signed
        if ($status['status'] === 'completed') {
            $this->handleCompletedSigning($requestId, $status);
        }
        
        return $status;
    }
    
    private function handleCompletedSigning($requestId, $status) {
        $request = $this->getRequest($requestId);
        
        // Download and store signed document
        $document = $this->provider->getSignedDocument($request['envelope_id']);
        $this->storeSignedDocument($requestId, $document);
        
        // Update request
        $this->db->where('id', $requestId)
            ->update('mod_esign_requests', [
                'completed_at' => $status['completed_at'],
                'signed_document_path' => $this->getSignedDocumentPath($requestId)
            ]);
        
        // Trigger webhooks
        $this->triggerSignedWebhook($requestId);
        
        // Log completion
        logActivity("Document signed successfully: {$request['envelope_id']}");
    }
    
    // Get signer signing URL
    public function getSignerUrl($requestId, $signerEmail) {
        $request = $this->getRequest($requestId);
        
        return $this->provider->getSignerUrl($request['envelope_id'], $signerEmail);
    }
    
    // Resend signature request
    public function resendRequest($requestId) {
        $request = $this->getRequest($requestId);
        $signers = json_decode($request['signers'], true);
        
        foreach ($signers as $signer) {
            // Resend to each signer
            $this->provider->resendEnvelope($request['envelope_id'], $signer['email']);
        }
        
        $this->db->where('id', $requestId)
            ->update('mod_esign_requests', [
                'resent_count' => $this->db->inc(1),
                'last_resent_at' => date('Y-m-d H:i:s')
            ]);
        
        return true;
    }
    
    // Void a pending signature request
    public function voidRequest($requestId, $reason) {
        $request = $this->getRequest($requestId);
        
        $this->provider->voidEnvelope($request['envelope_id'], $reason);
        
        $this->db->where('id', $requestId)
            ->update('mod_esign_requests', [
                'status' => 'voided',
                'voided_at' => date('Y-m-d H:i:s'),
                'void_reason' => $reason
            ]);
        
        return true;
    }
    
    private function notifySigners($requestId, $signers) {
        $request = $this->getRequest($requestId);
        $document = $this->getDocument($request['document_id']);
        
        foreach ($signers as $signer) {
            $signerUrl = $this->getSignerUrl($requestId, $signer['email']);
            
            sendEmail('signature_request', $signer['email'], [
                'document_name' => $document['name'],
                'signer_name' => $signer['name'],
                'sign_url' => $signerUrl,
                'expiry_days' => $this->config['expiry_days'] ?? 14
            ]);
        }
    }
    
    private function triggerSignedWebhook($requestId) {
        $request = $this->getRequest($requestId);
        
        // Trigger custom webhook
        $webhookUrl = $this->config['webhook_url'];
        
        if ($webhookUrl) {
            $this->sendWebhook($webhookUrl, [
                'event' => 'document.signed',
                'request_id' => $requestId,
                'document_id' => $request['document_id'],
                'signed_at' => $request['completed_at']
            ]);
        }
    }
}
```

### Signature Template Manager
```php
class SignatureTemplateManager {
    public function createTemplate($data) {
        $template = [
            'name' => $data['name'],
            'description' => $data['description'],
            'document_content' => $data['document_content'],
            'signature_fields' => json_encode($data['signature_fields']),
            'signers' => json_encode($data['signers']),
            'is_active' => 1,
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        return $this->db->insert('mod_esign_templates', $template);
    }
    
    public function getTemplate($templateId) {
        return $this->db->select(
            "SELECT * FROM mod_esign_templates WHERE id = ? AND is_active = 1",
            [$templateId]
        );
    }
    
    public function populateTemplate($templateId, $variables) {
        $template = $this->getTemplate($templateId);
        
        $content = $template['document_content'];
        
        // Replace variables
        foreach ($variables as $key => $value) {
            $content = str_replace('{{' . $key . '}}', $value, $content);
        }
        
        return [
            'name' => $this->interpolate($template['name'], $variables),
            'subject' => $this->interpolate($template['description'], $variables),
            'content' => $content,
            'signature_fields' => json_decode($template['signature_fields'], true),
            'signers' => $this->interpolateSigners($template['signers'], $variables)
        ];
    }
    
    private function interpolate($text, $variables) {
        foreach ($variables as $key => $value) {
            $text = str_replace('{{' . $key . '}}', $value, $text);
        }
        return $text;
    }
    
    private function interpolateSigners($signersJson, $variables) {
        $signers = json_decode($signersJson, true);
        
        return array_map(function($signer) use ($variables) {
            return [
                'email' => $this->interpolate($signer['email'], $variables),
                'name' => $this->interpolate($signer['name'], $variables),
                'order' => $signer['order']
            ];
        }, $signers);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_esign_requests (
    id INT AUTO_INCREMENT PRIMARY KEY,
    document_id INT NOT NULL,
    envelope_id VARCHAR(100),
    provider VARCHAR(50),
    status ENUM('draft', 'sent', 'delivered', 'signed', 'completed', 'voided', 'expired') DEFAULT 'draft',
    signers JSON,
    options JSON,
    sent_at DATETIME,
    delivered_at DATETIME,
    completed_at DATETIME,
    voided_at DATETIME,
    void_reason VARCHAR(255),
    signed_document_path VARCHAR(500),
    resent_count INT DEFAULT 0,
    last_resent_at DATETIME,
    created_by INT,
    created_at DATETIME,
    INDEX idx_document (document_id),
    INDEX idx_status (status),
    INDEX idx_envelope (envelope_id)
);

CREATE TABLE mod_esign_templates (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    document_content LONGTEXT,
    signature_fields JSON,
    signers JSON,
    variables JSON,
    is_active TINYINT(1) DEFAULT 1,
    created_at DATETIME,
    updated_at DATETIME
);

CREATE TABLE mod_esign_documents (
    id INT AUTO_INCREMENT PRIMARY KEY,
    request_id INT NOT NULL,
    original_path VARCHAR(500),
    signed_path VARCHAR(500),
    file_size INT,
    content_hash VARCHAR(64),
    signed_at DATETIME,
    signature_valid TINYINT(1),
    INDEX idx_request (request_id)
);

CREATE TABLE mod_esign_webhooks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    request_id INT NOT NULL,
    event_type VARCHAR(50),
    payload JSON,
    received_at DATETIME,
    processed TINYINT(1) DEFAULT 0,
    processed_at DATETIME
);
```

## Usage Examples

### Send Contract for Signature
```php
$manager = new WHMCSEsignManager('docusign');

// Create document
$document = [
    'id' => $contractId,
    'name' => 'Service Agreement.pdf',
    'subject' => 'Please sign your Service Agreement',
    'message' => 'Please review and sign the attached service agreement.',
    'content' => file_get_contents($contractPath)
];

$signers = [
    [
        'email' => 'client@example.com',
        'name' => 'John Doe',
        'order' => 1,
        'signature_fields' => [['page' => 1, 'x' => 100, 'y' => 700]]
    ],
    [
        'email' => 'legal@company.com',
        'name' => 'Company Legal',
        'order' => 2,
        'signature_fields' => [['page' => 2, 'x' => 100, 'y' => 700]]
    ]
];

$requestId = $manager->sendForSignature($document, $signers);
```

### Check Signature Status
```php
$manager = new WHMCSEsignManager();
$status = $manager->checkStatus($requestId);

echo "Status: {$status['status']}\n";
echo "Sent: {$status['sent_at']}\n";
echo "Completed: {$status['completed_at']}\n";
```

### Use Template for Signing
```php
$templateManager = new SignatureTemplateManager();
$template = $templateManager->populateTemplate($templateId, [
    'client_name' => 'John Doe',
    'contract_value' => '$1,000',
    'effective_date' => '2026-01-01'
]);

$signers = json_decode($template['signers'], true);
$requestId = $manager->sendForSignature($template, $signers);
```

## Best Practices

1. **Use reputable providers**: DocuSign, HelloSign, Adobe Sign are industry standards
2. **Maintain audit trail**: Log all signature events for legal compliance
3. **Store documents securely**: Use encrypted storage for signed documents
4. **Set expiration dates**: Prevent indefinite pending signatures
5. **Send reminders**: Automatically remind signers who haven't signed
6. **Handle voids gracefully**: Allow cancellation of pending requests
7. **Verify signatures**: Validate signature authenticity on completion