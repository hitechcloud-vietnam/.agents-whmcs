---
name: whmcs-invoice-discount
description: Apply discount to invoice in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, discount, billing]
---

# WHMCS Invoice Discount Workflow

## Purpose
Step-by-step guide for applying discounts to invoices in WHMCS.

## Prerequisites
- WHMCS admin access
- Invoice ID
- Discount authorization
- Discount code or manual discount

## Step 1: Access Discount Function
- Navigate to WHMCS Admin > Billing > Invoices
- Open target invoice
- Click "Apply Discount" button
- Or edit invoice to add discount line

## Step 2: Determine Discount Type
A. Percentage Discount:
   - X% off total or item
   - Common: 10%, 15%, 20%
   - Calculate from subtotal

B. Fixed Amount Discount:
   - $X off total or item
   - Common: $5, $10, $50
   - Subtract from subtotal

C. Promo Code Discount:
   - Valid discount code entry
   - Auto-calculate discount
   - Track usage

D. Manual Adjustment:
   - Custom discount amount
   - Requires authorization
   - Document reason

## Step 3: Configure Discount
- Select discount type
- Enter discount value
- Choose application scope:
  - Entire invoice
  - Specific line items
  - Specific products
- Set start/expiry if time-limited

## Step 4: Apply Tax Considerations
- Discount before tax (standard)
- Discount after tax (varies by region)
- TaxInclusive pricing
- Verify tax treatment

## Step 5: Preview Discount
- View original subtotal
- See discount amount
- Check new subtotal
- Verify tax recalculation

## Step 6: Apply Discount
- Click "Apply Discount"
- System calculates new total
- Updates invoice line items
- Records discount in notes

## Step 7: Post-Discount Actions
- Send updated invoice
- Log discount reason
- Track discount usage
- Update reports

## Discount Scenarios
1. Promotional: Black Friday sale
2. Loyalty: Long-term customer
3. Volume: Bulk order discount
4. Correction: Billing error
5. Retention: At-risk customer

## Discount Best Practices
- Document reason for discount
- Set approval limits
- Track discount effectiveness
- Monitor discount abuse
- Report discount costs

## Discount Code Features
- Single use or multi-use
- Percentage or fixed
- Minimum order value
- Specific products only
- Expiration date

## Related Workflows
- whmcs-coupon-creation
- whmcs-invoice-editing
- whmcs-discount-rules