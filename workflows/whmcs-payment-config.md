# WHMCS Payment Gateway Setup Workflow

## Purpose
Configure payment gateways for accepting payments

## Prerequisites
- WHMCS installed
- Admin access
- Merchant accounts

## Step 1: Navigate to Payment Gateways

Navigate to: Setup > Payments > Payment Gateways

## Step 2: Activate PayPal

1. Click "All Payment Gateways"
2. Find "PayPal" and click "Activate"
3. Configure:
   ```
   PayPal Email Address: your-paypal@email.com
   Payment Gateway Mode: Live
   ```

## Step 3: Activate Stripe

1. Activate "Stripe"
2. Configure:
   ```
   Publishable Key: pk_live_xxx
   Secret Key: sk_live_xxx
   Webhook Secret: whsec_xxx
   ```

## Step 4: Activate Bank Transfer

1. Activate "Bank Transfer / Direct Debit"
2. Configure:
   ```
   Bank Name: Your Bank Name
   Account Name: Your Company Name
   Account Number: XXXXXXXX
   Sort Code: XX-XX-XX
   SWIFT/BIC: XXXXXXXX
   IBAN: XXXXXXXXXXXXXX
   ```

## Step 5: Configure Offline Credit Card

Navigate to: Setup > Payments > Payment Gateways

1. Activate "Credit Card"
2. Configure merchant credentials

## Step 6: Set Payment Gateway Settings

Navigate to: Setup > Payments > Payment Gateway Settings

```
Allow Multiple Currencies: Yes
Auto-create Invoice After Order: Yes
Payment Gateway Order: [Drag to reorder]
```

## Step 7: Configure Invoice Settings

Navigate to: Setup > Payments > Invoice Settings

```
Auto Invoice Generation: Yes
Invoice Due After Days: 7
Invoice Starting Number: 1000
Invoice Prefix: INV-
```

## Step 8: Set Up Payment Methods

Navigate to: Setup > Payments > Payment Methods

```
Allowed Payment Methods: [Select defaults]
Show Payment Method Selection: Yes
Require Payment Method Selection: Yes
```

## Step 9: Test Payment Gateway

### Test Mode
1. Set gateway to "Test/Sandbox" mode
2. Place test order
3. Verify payment processes
4. Check admin notifications

### Live Mode
1. Switch to "Live" mode
2. Place small test transaction
3. Verify in merchant dashboard

## Step 10: Configure Payment Widgets

Navigate to: Setup > Payments > Payment Widgets

Enable inline payment forms for better conversion.

## Payment Gateway Reference

| Gateway | Type | Fees |
|---------|------|------|
| PayPal | Express Checkout | 2.9% + $0.30 |
| Stripe | Card Processing | 2.9% + $0.30 |
| Authorize.net | Card Processing | 3.5% + $0.15 |
| 2Checkout | Multi-gateway | 3.5% + $0.35 |

## Troubleshooting

### Payment Not Processing
- Verify API credentials
- Check webhook configuration
- Review error logs

### Webhook Issues (Stripe)
```bash
# Test webhook locally
stripe listen --forward-to localhost/whmcs/modules/gateways/callback/stripe.php
```
