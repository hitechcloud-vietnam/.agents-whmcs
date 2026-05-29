# WHMCS Identity Verification Module

## Overview
Identity verification module for KYC compliance.

## Module File: id_verification.php

```php
<?php
/**
 * WHMCS Identity Verification Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Database\Capsule;

class WHMCS_ID_Verification
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    /**
     * Submit verification request
     */
    public function submitVerification(int $userId, array $documents): int
    {
        return Capsule::table('mod_id_verifications')->insertGetId([
            'user_id' => $userId,
            'status' => 'pending',
            'submitted_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Get verification status
     */
    public function getStatus(int $userId): array
    {
        $verification = Capsule::table('mod_id_verifications')
            ->where('user_id', $userId)
            ->orderBy('submitted_at', 'desc')
            ->first();

        if (!$verification) {
            return ['status' => 'not_started'];
        }

        return [
            'status' => $verification->status,
            'submitted_at' => $verification->submitted_at,
            'verified_at' => $verification->verified_at,
        ];
    }

    /**
     * Verify document
     */
    public function verifyDocument(int $verificationId, bool $approved, ?string $reason): bool
    {
        $status = $approved ? 'approved' : 'rejected';

        Capsule::table('mod_id_verifications')
            ->where('id', $verificationId)
            ->update([
                'status' => $status,
                'verified_at' => date('Y-m-d H:i:s'),
                'rejection_reason' => $reason,
            ]);

        return true;
    }
}

add_hook('ClientAdd', 1, function($params) {
    if ($this->config['require_verification']) {
        Capsule::table('mod_id_verifications')->insert([
            'user_id' => $params['user_id'],
            'status' => 'required',
            'submitted_at' => date('Y-m-d H:i:s'),
        ]);
    }
});

function whmcs_id_verification_activate(): array
{
    try {
        if (!Capsule::schema()->hasTable('mod_id_verifications')) {
            Capsule::schema()->create('mod_id_verifications', function ($table) {
                $table->increments('id');
                $table->integer('user_id')->unsigned();
                $table->enum('status', ['pending', 'approved', 'rejected', 'required']);
                $table->text('rejection_reason')->nullable();
                $table->timestamp('submitted_at');
                $table->timestamp('verified_at')->nullable();
                $table->timestamp('created_at')->useCurrent();
                
                $table->index('user_id');
            });
        }
        return ['status' => 'success', 'description' => 'ID Verification activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_id_verification_deactivate(): array
{
    return ['status' => 'success', 'description' => 'ID Verification deactivated'];
}

function whmcs_id_verification_config(): array
{
    return [
        'require_verification' => ['FriendlyName' => 'Require Verification', 'Type' => 'yesno'],
        'auto_approve' => ['FriendlyName' => 'Auto Approve Low Risk', 'Type' => 'yesno'],
    ];
}
```

## Configuration File: config.php

```php
<?php
return [
    'require_verification' => false,
    'auto_approve' => false,
    'verification_provider' => 'manual',
];
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
