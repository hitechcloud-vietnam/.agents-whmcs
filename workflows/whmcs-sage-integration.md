# WHMCS Sage Integration Workflow

## Overview
This workflow implements Sage accounting integration for WHMCS.

## Prerequisites
- WHMCS with API access
- Sage account
- OAuth credentials

## Step-by-Step Process

### Step 1: Sage Integration
```php
<?php
// /includes/integrations/SageIntegration.php

class SageIntegration {
    private $accessToken;

    /**
     * Sync invoice to Sage
     */
    public function syncInvoice(int $invoiceId): array
    {
        $invoice = getInvoice($invoiceId);

        $data = [
            'InvoiceNumber' => 'WHMCS-' . $invoiceId,
            'CustomerName' => getClientName($invoice['userid']),
            'TotalAmount' => $invoice['total']
        ];

        return $this->apiCall('POST', '/invoices', $data);
    }

    private function apiCall(string $method, string $endpoint, array $data): array
    {
        $ch = curl_init(getConfig('sage_api_url') . $endpoint);

        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->accessToken,
                'Content-Type: application/json'
            ]
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}
```

## Related Workflows
- [WHMCS QuickBooks Integration](./whmcs-quickbooks-integration.md)
- [WHMCS Xero Integration](./whmcs-xero-integration.md)