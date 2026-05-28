# WHMCS Affiliate Tracking Module

Advanced affiliate tracking with commission tiers, referral tracking, and payout management.

## Features

- Affiliate registration
- Click tracking
- Referral tracking
- Commission calculation
- Tiered commissions
- Payout management
- Statistics

## Installation

Copy module to `/path/to/whmcs/modules/servers/affiliatetracking/` and activate.

## Usage

```php
// Register affiliate
$affiliate = affiliatetracking_RegisterAffiliate($userId, 15); // 15% commission

// Get affiliate
$aff = affiliatetracking_GetAffiliate($userId);

// Get by code
$aff = affiliatetracking_GetAffiliateByCode('AFF12345678');

// Track click
affiliatetracking_TrackClick($affiliateId);

// Record referral
affiliatetracking_RecordReferral($affiliateId, $referredUserId, 99.99, $invoiceId);

// Get referrals
$referrals = affiliatetracking_GetReferrals($affiliateId, 'pending');

// Get statistics
$stats = affiliatetracking_GetStatistics($affiliateId);
// Returns: total_clicks, total_referrals, conversion_rate, total_earned, pending_balance, total_paid

// Set commission rate
affiliatetracking_SetCommissionRate($affiliateId, 20);

// Create payout
affiliatetracking_CreatePayout($affiliateId, 100.00, 'paypal');
```

## API Functions

| Function | Description |
|----------|-------------|
| `affiliatetracking_RegisterAffiliate()` | Register new affiliate |
| `affiliatetracking_GetAffiliate()` | Get affiliate by user |
| `affiliatetracking_GetAffiliateByCode()` | Get by affiliate code |
| `affiliatetracking_TrackClick()` | Track referral click |
| `affiliatetracking_RecordReferral()` | Record sale referral |
| `affiliatetracking_GetReferrals()` | Get affiliate referrals |
| `affiliatetracking_GetStatistics()` | Get affiliate stats |
| `affiliatetracking_SetCommissionRate()` | Update commission rate |
| `affiliatetracking_CreatePayout()` | Create payout |
