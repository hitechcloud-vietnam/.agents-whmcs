---
name: whmcs-invoice-tax-adjust
description: Adjust tax on invoice in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, tax, adjustment]
---

# WHMCS Invoice Tax Adjustment Workflow

## Purpose
Step-by-step guide for adjusting taxes on invoices in WHMCS.

## Prerequisites
- WHMCS admin access
- Invoice ID
- Tax adjustment authorization
- Tax rule understanding

## Step 1: Access Tax Adjustment
- Navigate to WHMCS Admin > Billing > Invoices
- Open target invoice
- Click "Edit" to modify
- Locate tax configuration

## Step 2: Verify Tax Adjustment Reason
- Tax rate changed
- Client exemption status change
- Tax jurisdiction correction
- Tax rule misconfiguration
- Manual adjustment needed

## Step 3: Review Current Tax
- View applied tax line items
- Check tax rate percentage
- Verify tax calculation basis
- Identify tax rules applied

## Step 4: Configure Tax Adjustment
Options:
A. Change Tax Rate:
   - Select different tax rule
   - Enter new rate percentage
   - Apply to all items

B. Add Additional Tax:
   - Add second tax (compound)
   - Configure compound calculation
   - Set appropriate rate

C. Remove Tax:
   - Apply exemption
   - Set tax rate to 0%
   - Document exemption reason

D. Manual Adjustment:
   - Enter specific tax amount
   - Override calculated tax
   - Document justification

## Step 5: Apply Tax Changes
- Select items to adjust
- Apply new tax rate
- Preview new tax amount
- Verify calculation accuracy

## Step 6: Save Tax Adjustment
- Review total changes
- Check new invoice total
- Click "Save Changes"
- System updates tax calculations

## Step 7: Post-Adjustment Actions
- Send updated invoice to client
- Document tax adjustment reason
- Update tax records
- Log adjustment in audit trail

## Tax Adjustment Scenarios
1. VAT rate change mid-period
2. Client tax exemption applied
3. Wrong tax jurisdiction
4. Compound tax configuration
5. Quarterly tax adjustment

## Tax Compliance Notes
- Maintain tax documentation
- Follow local tax regulations
- Track adjustment reasons
- Report tax adjustments
- Keep audit trail

## Related Workflows
- whmcs-invoice-editing
- whmcs-tax-rule-setup
- whmcs-invoice-payment