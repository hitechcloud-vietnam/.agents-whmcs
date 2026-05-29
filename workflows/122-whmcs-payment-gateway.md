---
name: whmcs-payment-gateway
description: Payment gateway integration in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, gateway, integration]
---

# WHMCS Payment Gateway Integration Workflow

## Purpose
Step-by-step guide for integrating payment gateways in WHMCS.

## Prerequisites
- WHMCS admin access
- Gateway account credentials
- SSL certificate configured
- Developer access (for custom)

## Step 1: Select Payment Gateway
- Review WHMCS supported gateways
- Popular options:
  - Stripe
  - PayPal
  - Authorize.net
  - 2Checkout
  - Braintree
- Check gateway compatibility

## Step 2: Create Gateway Account
- Sign up with gateway provider
- Complete merchant application
- Get API credentials:
  - API Key/Secret
  - Merchant ID
  - Webhook URL
- Enable required features

## Step 3: Install Gateway Module
- Navigate to Setup > Payments > Payment Gateways
- Click "Manage Existing Gateways"
- Install new gateway module
- Activate module

## Step 4: Configure Gateway Settings
- Enter credentials:
  - API Key
  - Merchant ID
  - Secret Key
- Set configuration:
  - Transaction mode (Live/Test)
  - Currency support
  - Fee handling
  - Webhook URL

## Step 5: Configure Webhook
- Set WHMCS webhook URL
- Configure webhook events:
  - Payment success
  - Payment failed
  - Refund processed
  - Chargeback received
- Test webhook connectivity

## Step 6: Test Gateway Integration
- Enable test/sandbox mode
- Process test payment
- Verify transaction recording
- Test refund processing
- Check webhook delivery

## Step 7: Go Live
- Switch to live credentials
- Disable test mode
- Verify all features work
- Monitor initial transactions
- Set up monitoring

## Custom Gateway Development
For custom gateways:
1. Create module in /whmcs/includes/gateways/
2. Implement required functions
3. Follow gateway API documentation
4. Test thoroughly before deployment

## Gateway Comparison
| Gateway | Fees | Features | Integration |
|---------|------|----------|-------------|
| Stripe | 2.9%+ | Full featured | API |
| PayPal | 2.9%+ | Widely used | SDK |
| Authorize | 2.9%+ | Stable | API |

## Related Workflows
- whmcs-payment-accept
- whmcs-gateway-module-development
- whmcs-gateway-from-scratch