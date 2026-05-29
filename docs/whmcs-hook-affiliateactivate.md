# WHMCS AffiliateActivate Hook Reference

## Overview

The `AffiliateActivate` hook fires when an affiliate account is activated in WHMCS. This hook triggers when a client signs up as an affiliate or when an admin enables an affiliate account.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `affiliateid` | int | The affiliate ID |
| `userid` | int | The client/user ID |
| `code` | string | Unique affiliate code |
| `rate` | float | Commission rate |
| `money` | float | Payment threshold |

## Example Implementation

```php
<?php
add_hook('AffiliateActivate', 1, function(array $params) {
    // Log affiliate activation
    logActivity("Affiliate activated: ID {$params['affiliateid']} for user {$params['userid']}");
    
    // Send welcome email
    $client = getClientsDetails($params['userid']);
    sendTemplatedEmail('Affiliate Welcome', $client['email'], [
        'affiliate_id' => $params['affiliateid'],
        'referral_code' => $params['code']
    ]);
    
    return $params;
});
```

## Onboarding Workflow

```php
<?php
add_hook('AffiliateActivate', 1, function(array $params) {
    // 1. Generate referral link
    $referralLink = "https://yoursite.com/?ref=" . $params['code'];
    update_query('tblaffiliates', [
        'referral_link' => $referralLink
    ], ['id' => $params['affiliateid']]);
    
    // 2. Set up affiliate dashboard access
    createAffiliateDashboard($params['affiliateid']);
    
    // 3. Assign welcome bonus
    $welcomeBonus = getSetting('affiliate_welcome_bonus');
    if ($welcomeBonus > 0) {
        insert_query('tblaffiliates', [
            'balance' => $welcomeBonus
        ], ['id' => $params['affiliateid']]);
        
        // Add transaction record
        insert_query('tblaffiliatestransactions', [
            'affiliateid' => $params['affiliateid'],
            'date' => date('Y-m-d H:i:s'),
            'amount' => $welcomeBonus,
            'type' => 'Welcome Bonus',
            'status' => 'Completed'
        ]);
    }
    
    // 4. Create onboarding task list
    createAffiliateOnboardingTasks($params['affiliateid']);
    
    // 5. Add to affiliate marketing segment
    addToMarketingSegment($params['userid'], 'affiliates');
    
    return $params;
});
```

## Default Configuration

```php
<?php
add_hook('AffiliateActivate', 1, function(array $params) {
    // 1. Set default commission tier
    $defaultTier = getDefaultAffiliateTier();
    update_query('tblaffiliates', [
        'tier_id' => $defaultTier['id'],
        'commission_rate' => $defaultTier['rate']
    ], ['id' => $params['affiliateid']]);
    
    // 2. Set default payment method
    update_query('tblaffiliates', [
        'paymentmethod' => 'default'
    ], ['id' => $params['affiliateid']]);
    
    // 3. Configure notification preferences
    insert_query('tbl_affiliate_notifications', [
        'affiliate_id' => $params['affiliateid'],
        'email_referrals' => 1,
        'email_payments' => 1,
        'email_threshold' => 50.00
    ]);
    
    // 4. Create tracking for analytics
    setupAffiliateAnalytics($params['affiliateid'], $params['code']);
    
    return $params;
});
```

## Integration with External Systems

```php
<?php
add_hook('AffiliateActivate', 1, function(array $params) {
    // 1. Sync to email marketing platform
    $client = getClientsDetails($params['userid']);
    addToEmailList($client['email'], 'affiliate_program', [
        'affiliate_id' => $params['affiliateid'],
        'signup_date' => date('Y-m-d')
    ]);
    
    // 2. Integrate with affiliate network (e.g., Refersion)
    syncToAffiliateNetwork($params);
    
    // 3. Set up webhook for new affiliates
    sendWebhook('affiliate.activated', [
        'affiliate_id' => $params['affiliateid'],
        'user_id' => $params['userid'],
        'referral_code' => $params['code'],
        'commission_rate' => $params['rate']
    ]);
    
    // 4. Track in analytics
    trackAnalyticsEvent('affiliate_signup', [
        'affiliate_id' => $params['affiliateid'],
        'source' => 'whmcs'
    ]);
    
    return $params;
});
```

## Use Cases

- **Onboarding**: Welcome sequence and setup
- **Bonuses**: Welcome bonuses or credits
- **Integrations**: Sync with external platforms
- **Analytics**: Track affiliate signups
- **Configuration**: Set default preferences

## Notes

- Runs when affiliate status changes to Active
- Use `AffiliateReject` for rejected signups
- Consider GDPR for marketing communications
- Track affiliate activity from signup

## Related Hooks

- `AffiliateActivate` - Affiliate activation
- `AffiliateCommissions` - Commission tracking
- `AffiliatePayout` - Affiliate payments

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Affiliate Configuration](../whmcs-affiliate-setup.md)