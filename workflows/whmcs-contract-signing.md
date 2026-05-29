# WHMCS Contract Signing Workflow

## Overview
This workflow automates electronic contract signing in WHMCS.

## Prerequisites
- E-signature service integration
- Contract templates

## Step-by-Step Guide

### Step 1: Configure E-Signature Service
```php
// lib/EsignService.php
class EsignService
{
    private string $apiKey;
    private string $endpoint;

    public function __construct()
    {
        $config = getGatewayVariables('esign');
        $this->apiKey = $config['api_key'];
        $this->endpoint = $config['endpoint'];
    }

    public function createSignatureRequest(array $params): array
    {
        $response = $this->request('POST', '/signatures', [
            'document' => base64_encode($params['document']),
            'signers' => [
                [
                    'email' => $params['email'],
                    'name' => $params['name'],
                ],
            ],
            'subject' => $params['subject'],
            'message' => $params['message'],
        ]);

        return [
            'signature_id' => $response['id'],
            'sign_url' => $response['signing_url'],
        ];
    }

    public function getSignatureStatus(string $signatureId): string
    {
        $response = $this->request('GET', "/signatures/$signatureId");
        return $response['status'];
    }

    public function downloadSignedDocument(string $signatureId): string
    {
        $response = $this->request('GET', "/signatures/$signatureId/document");
        return base64_decode($response['document']);
    }
}
```

### Step 2: Create Contract Hook
```php
add_hook('OrderFulfillment', 1, function($vars) {
    $orderId = $vars['orderId'];
    $clientId = $vars['userId'];
    
    $client = \WHMCS\Database\Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    $esign = new EsignService();
    
    $contract = $esign->createSignatureRequest([
        'document' => file_get_contents(__DIR__ . '/templates/service_contract.pdf'),
        'email' => $client->email,
        'name' => $client->firstname . ' ' . $client->lastname,
        'subject' => 'Please sign your service contract',
        'message' => 'Your service contract is ready for signing.',
    ]);
    
    // Store contract info
    \WHMCS\Database\Capsule::table('mod_yourmodule_contracts')->insert([
        'order_id' => $orderId,
        'client_id' => $clientId,
        'signature_id' => $contract['signature_id'],
        'sign_url' => $contract['sign_url'],
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Send signing link to client
    send_email($client->email, 'contract_pending', [
        'sign_url' => $contract['sign_url'],
    ]);
});
```

## Contract Signing Checklist

### Setup
- [ ] E-sign service configured
- [ ] Contract templates ready
- [ ] API credentials set

### Signing
- [ ] Contract sent
- [ ] Signer notified
- [ ] Status tracked

### Completion
- [ ] Document signed
- [ ] Download stored
- [ ] Order completed
