# WHMCS Referral Credit Workflow

## Overview
Automated referral system that credits both referrer and referee.

## Prerequisites
- WHMCS v8.0+
- Credit system

## Step-by-Step Guide

### Step 1: Referral Configuration
```php
<?php
class ReferralService
{
    protected float $referrerCredit = 20.00;
    protected float $refereeCredit = 10.00;

    public function generateReferralCode(int $clientId): string
    {
        $code = strtoupper(substr(md5($clientId . time()), 0, 8));
        
        \WHMCS\Database\Capsule::table('mod_yourmodule_referrals')->insert([
            'client_id' => $clientId,
            'referral_code' => $code,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return $code;
    }

    public function processReferral(int $referrerId, int $refereeId): void
    {
        // Credit referrer
        addCredit($referrerId, $this->referrerCredit, 'Referral bonus');
        
        // Credit referee
        addCredit($refereeId, $this->refereeCredit, 'Welcome bonus via referral');

        // Log referral
        \WHMCS\Database\Capsule::table('mod_yourmodule_referral_history')->insert([
            'referrer_id' => $referrerId,
            'referee_id' => $refereeId,
            'referrer_credit' => $this->referrerCredit,
            'referee_credit' => $this->refereeCredit,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

### Step 2: Register Referral Hook
```php
add_hook('ClientCreate', 1, function($vars) {
    if (!isset($_COOKIE['referral_code'])) {
        return;
    }

    $code = $_COOKIE['referral_code'];
    $referrer = \WHMCS\Database\Capsule::table('mod_yourmodule_referrals')
        ->where('referral_code', $code)->first();

    if ($referrer) {
        $referral = new \Vendor\Module\ReferralService();
        $referral->processReferral($referrer->client_id, $vars['userid']);
    }
});
```

## Checklist
- Referral codes generated
- Credits applied on signup
- History tracked
- Reports available
