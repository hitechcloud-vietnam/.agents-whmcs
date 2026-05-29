# WHMCS Domain Pricing Setup Workflow

## Purpose
Configure domain pricing for registration, transfer, and renewal across all TLDs (Top Level Domains).

## Prerequisites
- WHMCS installation with domain module
- Registrar modules configured
- Pricing strategy defined

## Step-by-Step Process

### Step 1: Access Domain Pricing
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > Domain Pricing`
3. Review current pricing structure

### Step 2: Configure Registrar Pricing
1. Set up registrar costs:
   - Enter cost per TLD from registrar
   - Set markup percentage
   - Configure currency conversion
   - Set renewal pricing structure
2. Import registrar pricing

### Step 3: Set Registration Pricing
1. Configure registration fees:
   - .com registration price
   - .net registration price
   - .org registration price
   - Country-code TLDs pricing
   - Premium domain pricing
2. Set billing term pricing (1yr, 2yr, 5yr, 10yr)

### Step 4: Configure Transfer Pricing
1. Set transfer fees:
   - Transfer cost per TLD
   - Transfer markup
   - Include 1-year renewal option
   - Premium transfer pricing
2. Configure transfer restrictions

### Step 5: Set Renewal Pricing
1. Configure renewal fees:
   - Standard renewal price
   - Premium domain renewal
   - Grace period pricing
   - Redemption period pricing
2. Set auto-renewal pricing

### Step 6: Configure Bulk Pricing
1. Set bulk discount pricing:
   - Multi-year registration discounts
   - Bulk transfer pricing
   - Quantity discounts
2. Set up promotional pricing

### Step 7: Set Up Premium Domains
1. Configure premium domain handling:
   - Premium pricing tiers
   - Premium registration markup
   - Premium transfer pricing
   - Premium renewal pricing
2. Set up premium domain display

### Step 8: Configure Addon Pricing
1. Set addon services:
   - WHOIS privacy pricing
   - DNS management pricing
   - Email forwarding pricing
   - ID protection pricing
2. Set bundled addon pricing

### Step 9: Set Up Group Pricing
1. Configure client group pricing:
   - VIP client discounts
   - Reseller pricing
   - Affiliate commissions
2. Set geographic pricing

### Step 10: Review and Test Pricing
1. Test pricing calculations:
   - Registration pricing
   - Transfer pricing
   - Renewal pricing
   - Addon pricing
2. Verify currency display

## Verification Checklist
- [ ] All TLDs have pricing
- [ ] Registration prices display correctly
- [ ] Transfer pricing accurate
- [ ] Renewal pricing shows
- [ ] Group pricing applies

## Related Workflows
- whmcs-tld-import
- whmcs-domain-registration-flow
- whmcs-domain-transfer-flow
- whmcs-whois-privacy-setup

## Domain Pricing Best Practices
- Competitive pricing strategy
- Clear markup structure
- Regular pricing review
- Bulk discount incentives
- Premium domain handling