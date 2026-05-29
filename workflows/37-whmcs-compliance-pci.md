# WHMCS PCI DSS Compliance Workflow

## Overview
This workflow ensures payment card data handling complies with PCI DSS requirements.

## Step 1: PCI Compliance Service

```php
<?php
// src/Service/PciComplianceService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class PciComplianceService
{
    private $auditLog;

    public function __construct()
    {
        $this->auditLog = new AuditLogService();
    }

    /**
     * Check if card data storage is compliant
     */
    public function checkCardDataStorage(): array
    {
        $findings = [];

        // Check for raw card data in database
        $rawCardData = Capsule::table('tblcreditcards')
            ->where('card_number', '!=', '')
            ->count();

        if ($rawCardData > 0) {
            $findings[] = [
                'severity' => 'critical',
                'issue' => 'Raw card data found in database',
                'recommendation' => 'Migrate to tokenization immediately'
            ];
        }

        // Check for card data in logs
        $cardInLogs = Capsule::table('mod_activity_logs')
            ->where('data', 'REGEXP', '[0-9]{13,16}')
            ->count();

        if ($cardInLogs > 0) {
            $findings[] = [
                'severity' => 'high',
                'issue' => 'Potential card data in logs',
                'recommendation' => 'Review and sanitize log entries'
            ];
        }

        return [
            'compliant' => empty($findings),
            'findings' => $findings,
            'checked_at' => date('Y-m-d H:i:s')
        ];
    }

    /**
     * Tokenize card data for storage
     */
    public function tokenizeCardData(int $clientId, array $cardData): string
    {
        // Send to payment gateway for tokenization
        $gateway = new PaymentGatewayService();
        $token = $gateway->tokenizeCard($cardData);

        // Store only token, not card data
        Capsule::table('mod_payment_tokens')->insert([
            'client_id' => $clientId,
            'token' => $token,
            'last_four' => substr($cardData['number'], -4),
            'exp_month' => $cardData['exp_month'],
            'exp_year' => $cardData['exp_year'],
            'card_type' => $this->detectCardType($cardData['number']),
            'created_at' => date('Y-m-d H:i:s')
        ]);

        $this->auditLog->log('tokenize', 'pci', [
            'client_id' => $clientId,
            'last_four' => substr($cardData['number'], -4)
        ]);

        return $token;
    }

    /**
     * Process payment using token
     */
    public function processTokenizedPayment(int $clientId, float $amount, string $invoiceId): array
    {
        $token = Capsule::table('mod_payment_tokens')
            ->where('client_id', $clientId)
            ->orderBy('created_at', 'desc')
            ->first();

        if (!$token) {
            return ['success' => false, 'error' => 'No payment method on file'];
        }

        $gateway = new PaymentGatewayService();
        $result = $gateway->chargeToken($token->token, $amount);

        if ($result['success']) {
            $this->auditLog->logPayment($invoiceId, 'token_charge', [
                'token_id' => $token->id,
                'amount' => $amount
            ]);
        }

        return $result;
    }

    /**
     * Delete stored card token
     */
    public function deletePaymentMethod(int $tokenId): void
    {
        $token = Capsule::table('mod_payment_tokens')->where('id', $tokenId)->first();

        if ($token) {
            // Notify gateway to delete token
            $gateway = new PaymentGatewayService();
            $gateway->deleteToken($token->token);

            Capsule::table('mod_payment_tokens')->where('id', $tokenId)->delete();

            $this->auditLog->log('delete', 'pci', [
                'token_id' => $tokenId
            ]);
        }
    }

    private function detectCardType(string $number): string
    {
        if (preg_match('/^4/', $number)) return 'Visa';
        if (preg_match('/^5[1-5]/', $number)) return 'Mastercard';
        if (preg_match('/^3[47]/', $number)) return 'Amex';
        if (preg_match('/^6(?:011|5)/', $number)) return 'Discover';
        return 'Unknown';
    }

    /**
     * Generate PCI compliance report
     */
    public function generateComplianceReport(): array
    {
        return [
            'scanned_at' => date('Y-m-d H:i:s'),
            'card_storage_check' => $this->checkCardDataStorage(),
            'tokenized_methods' => Capsule::table('mod_payment_tokens')->count(),
            'active_tokens' => Capsule::table('mod_payment_tokens')
                ->where('deleted_at', null)->count(),
            'compliant' => $this->checkCardDataStorage()['compliant']
        ];
    }
}
```

## Verification Checklist

- [ ] Card data storage compliant
- [ ] Tokenization implemented
- [ ] Token deletion working
- [ ] Audit logging for payments
- [ ] Compliance report generating
- [ ] PCI requirements met
