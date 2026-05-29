# WHMCS Affiliate Program Setup Workflow

## Purpose
Configure WHMCS affiliate/referral program

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Navigate to Affiliate Settings

Navigate to: Setup > Affiliates

## Step 2: Enable Affiliate System

```
Enable Affiliates: Yes
Allow Affiliate Signups: Yes
Require Approval: No
```

## Step 3: Configure Commission Settings

Navigate to: Setup > Affiliates > Affiliate Settings

### Commission Structure
```
Commission Type: Percentage / Fixed Amount
Commission Rate: 5%
Commission on: First Order Only / All Orders
Include Renewals: Yes
Include Addons: Yes
Include Upgrades: Yes
```

### Calculation Settings
```
Calculation Method: After Due Date / After Payment
Waiting Period: 30 days
Revoke on Refund: Yes
Revoke on Cancellation: Yes
```

## Step 4: Set Up Affiliate Tiers

Navigate to: Setup > Affiliates > Tiers

### Create Tier
1. Click "Add Tier"
2. Configure:
   ```
   Tier Name: Silver Partner
   Minimum Referrals: 5
   Commission Rate: 7%
   ```
3. Save

### Default Tiers
```
Bronze: 0-4 referrals (5%)
Silver: 5-14 referrals (7%)
Gold: 15-29 referrals (10%)
Platinum: 30+ referrals (15%)
```

## Step 5: Configure Payout Settings

Navigate to: Setup > Affiliates > Payout Settings

```
Minimum Payout: $50
Payout Methods:
  - PayPal
  - Bank Transfer
  - Account Credit
Auto Payout: No
Payout Request Frequency: Monthly
Processing Fee: $0.00 / 2%
```

## Step 6: Set Up Affiliate Tracking

### Tracking Settings
```
Cookie Duration: 365 days
Attribution: First Touch / Last Touch
Tracking Method: Browser Cookie
```

### Referral Links
```
Referral URL Format: https://domain.com/aff.php?aff=X
Clean URLs: Enabled
```

## Step 7: Configure Affiliate Portal

Navigate to: Setup > Affiliates > Portal Settings

### Dashboard Features
```
Show Statistics: Yes
Show Referral Links: Yes
Show Commissions: Yes
Show Payout History: Yes
Show Traffic Stats: Yes
Show Conversions: Yes
```

### Affiliate Resources
```
Show Marketing Materials: Yes
Include Banners: Yes
Include Email Templates: Yes
```

## Step 8: Set Up Affiliate Notifications

Navigate to: Setup > Affiliates > Notifications

```
New Referral: Notify affiliate
Commission Earned: Notify affiliate
Payout Processed: Notify affiliate
Tier Upgrade: Notify affiliate
```

## Step 9: Configure Affiliate Products

Navigate to: Setup > Affiliates > Product Restrictions

```
Commission Excluded Products:
  - Free products
  - Discounted items
  - Custom products
```

## Step 10: Create Affiliate Terms

Navigate to: Setup > Affiliates > Terms

Create terms page:
```
Commission Rules:
- Commission is paid after payment clears
- Refunded orders forfeit commission
- Self-referrals not allowed
- Fraudulent activity results in termination
```

## Step 11: Set Up Affiliate Landing Page

Navigate to: Configuration > System Settings > Custom Pages

Create affiliate program landing page with:
- Program benefits
- Commission structure
- Sign-up form
- Testimonials

## Step 12: Test Affiliate System

### Test Flow
1. Register as affiliate
2. Get referral link
3. Place test order using link
4. Verify tracking
5. Check commission recorded
6. Test payout request

## Affiliate Checklist

- [ ] Affiliate system enabled
- [ ] Commission settings configured
- [ ] Tiers created
- [ ] Payout settings set
- [ ] Tracking configured
- [ ] Portal configured
- [ ] Notifications enabled
- [ ] Product restrictions set
- [ ] Terms created
- [ ] System tested
