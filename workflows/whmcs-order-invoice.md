# WHMCS Order Invoice Workflow

## Purpose
Step-by-step guide for generating order invoices in WHMCS.

## Prerequisites
- Order created
- Invoice settings configured
- Billing cycle defined
- Payment method selected

## Workflow Steps

### Step 1: Invoice Generation Trigger
- Order created
- Automatic invoice generation
- Manual invoice request
- Renewal invoice trigger

### Step 2: Order Review
- Review order items
- Verify pricing
- Check quantities
- Validate configurations

### Step 3: Pricing Calculation
- Calculate item totals
- Apply discounts
- Calculate taxes
- Determine final amount

### Step 4: Invoice Creation
- Generate invoice record
- Assign invoice number
- Set invoice date
- Configure due date

### Step 5: Line Items
- Add order items as lines
- Include descriptions
- Set quantities
- Apply pricing

### Step 6: Tax Calculation
- Determine tax location
- Apply tax rules
- Calculate tax amount
- Add tax line

### Step 7: Discount Application
- Apply promo codes
- Apply discounts
- Calculate savings
- Update totals

### Step 8: Payment Terms
- Set due date
- Configure late fees
- Set payment terms
- Configure reminders

### Step 9: Invoice Delivery
- Send to client
- Generate PDF
- Set email template
- Track delivery

### Step 10: Payment Processing
- Process payment
- Update invoice status
- Record transaction
- Update order status

## Invoice Types
- Order invoice
- Renewal invoice
- Manual invoice
- Credit memo
- Proforma invoice

## Verification Checklist
- [ ] Invoice generated
- [ ] Items correct
- [ ] Taxes calculated
- [ ] Sent to client
- [ ] Payment processed

## Related Workflows
- whmcs-order-payment
- whmcs-order-refund
- whmcs-new-order-flow