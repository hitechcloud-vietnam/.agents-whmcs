# WHMCS Account Verification Workflow

## Overview
This workflow implements account verification for WHMCS.

## Prerequisites
- WHMCS with verification capabilities
- Email/SMS verification systems
- Admin access for verification settings

## Step-by-Step Process

### Step 1: Create Verification Manager
```php
<?php
// /includes/verification/AccountVerificationManager.php

class AccountVerificationManager {
    /**
     * Send verification code
     */
    public function sendVerification(int $clientId, string $method = 'email'): array
    {
        $code = $this->generateCode();

        Capsule::table('mod_verification_codes')->insert([
            'client_id' => $clientId,
            'code' => password_hash($code, PASSWORD_DEFAULT),
            'method' => $method,
            'expires_at' => date('Y-m-d H:i:s', strtotime('+15 minutes')),
            'created_at' => date('Y-m-d H:i:s')
        ]);

        if ($method === 'email') {
            sendEmail($clientId, 'Verification Code', ['code' => $code]);
        } elseif ($method === 'sms') {
            sendSMS($clientId, "Your verification code is: {$code}");
        }

        return ['success' => true, 'expires_in' => 900];
    }

    /**
     * Verify code
     */
    public function verifyCode(int $clientId, string $code): bool
    {
        $record = Capsule::table('mod_verification_codes')
            ->where('client_id', $clientId)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->where('verified', 0)
            ->orderBy('created_at', 'DESC')
            ->first();

        if ($record && password_verify($code, $record->code)) {
            Capsule::table('mod_verification_codes')
                ->where('id', $record->id)
                ->update(['verified' => 1, 'verified_at' => date('Y-m-d H:i:s')]);

            return true;
        }

        return false;
    }

    private function generateCode(): string
    {
        return str_pad((string)random_int(0, 999999), 6, '0', STR_PAD_LEFT);
    }
}
```

## Related Workflows
- [WHMCS Two-Factor Authentication](./whmcs-two-factor-auth.md)
- [WHMCS Security Scan](./whmcs-security-scan.md)